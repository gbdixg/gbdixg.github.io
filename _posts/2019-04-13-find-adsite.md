---
id: 18
title: Find a computer's Active Directory Site and Subnet
date: 2019-04-13T17:22:14+00:00
author: Geoff
description: Use PowerShell to determine the AD Site and Subnet of a Windows computer
layout: post
permalink: /2019/04/13/find-adsite
image: assets/images/thumb-find-adsite.png
comments: true
featured: false
hidden: false
categories: [ activedirectory, powershell ]
tags:
  - activedirectory
  - powershell
  - Level200
---
Active Directory Sites represent locations with good network connectivity. They are often created for each company office or a group of offices in a metropolitan area. AD Sites are used to generate the replication topology between Domain Controllers, and to help domain member computers locate close services - such as Domain Controllers or DFS share replicas.

Active Directory subnets define the parts of the network that are included in an AD Site. A domain member will have an IP address that should fall within a defined AD subnet. The subnet will be assigned to an AD Site, making the domain member part of that AD Site.

AD Site membership is not fixed. Laptops can move between Sites when they roam to another location.

> An incorrect or undefined AD subnet can lead to slow logon times and slow access to DFS shares. A domain member will use any server that responds when it isn't in a defined subnet.

The **Find-ADSite** PowerShell function below will return the AD Site and Subnet for a computer or IP Address.
If your company populates subnet descriptions with useful information, it can also identify information such as the specific Office or floor.

![Find-ADSite](/assets/images/find-adsite1.png){:class="img-responsive"}

```powershell
Function Find-ADSite {
    <#
	.Synopsis
		Used to get the Active Directory subnet and the site it is assigned to for a Windows computer/IP address

	.Description
		Requires only standard user read access to AD and can determine the ADSite for a local or remote computer

	.PARAMETER  IPAddress
		Specifies the IP Address for the subnet/site lookup in as a .NET System.Net.IPAddress

		When this parameter is used, the computername is not specified.

	.PARAMETER  Computername
		Specifies a computername for the subnet/site lookup.
		The computername is resolved to an IP address before performing the subnet query.

		Defaults to %COMPUTERNAME%

		When this parameter is used, the IPAddress and IP are not specified.

	.PARAMETER  DC
		A specific domain controller in the current users domain for the subnet query
		If not specified, standard DC locator methods are used.

	.PARAMETER  AllMatches
		A switch parameter that causes the subnet query to return all matching subnets in AD
		This is not normally used as the default behaviour (only the most specific match is returned) is usually prefered.
		This switch will include "catch-all" subnets that may be defined to accomodate missing subnets

	.Example
		PS C:\>Find-ADSite -ComputerName PC123456789

		ADSubnetName : 162.26.192.128/25
		IPAddress    : 162.26.192.151
		ComputerName : PC123456789
		ADSite       : EULON01
		ADSubnetDesc : GB-LONDON

	.Notes
		Version:        1.1
			Removed script block and parallel parallel support

#>
    [CmdletBinding(DefaultParameterSetName = "byHost")]
    Param(
        [Parameter(Position = 0, ValueFromPipeline = $true, ValueFromPipelineByPropertyName = $True, ParameterSetName = "byHost")]
        [string]$ComputerName = $Env:COMPUTERNAME
        ,
        [Parameter(Position = 0, ValueFromPipeline = $true, ValueFromPipelineByPropertyName = $True, Mandatory = $True, ParameterSetName = "byIPAddress")]
        [System.Net.IPAddress]$IPAddress
        ,
        [Parameter(Position = 1)]
        [string]$DC
        ,
        [Parameter()]
        [switch]$AllMatches

    )
    BEGIN {	}
    PROCESS {

        switch ($pscmdlet.ParameterSetName) {

            "byHost" {
                try {
                    $Resolved = [system.net.dns]::GetHostByName($Computername)
                    [System.Net.IPAddress]$IP = ($Resolved.AddressList)[0] -as [System.Net.IPAddress]
                }catch{
                    Write-Warning "$ComputerName :: Unable to resolve name to an IP Address"
                    $IP = $Null
                }
            }

            "byIPAddress" {
                try {
                    $Resolved = [system.net.dns]::GetHostByAddress($IPAddress)
                    $ComputerName = $Resolved.HostName
                } catch {
                    # Write-Warning "$IP :: Could not be resolved to a hostname"
                    $ComputerName = "Unable to resolve"
                }

                $IP = $IPAddress
            }

        }#switch

        if ($IP) {
            # The following maths loops over all the possible subnet mask lengths
            # The masks are converted into the number of Bits to allow conversion to CIDR format

            [psobject[]]$MatchedSubnets = @()

            For ($bit = 30 ; $bit -ge 1; $bit--) {

                [int]$octet = [math]::Truncate(($bit - 1 ) / 8)
                $net = [byte[]]@()

                for ($o = 0; $o -le 3; $o++) {
                    $ba = $ip.GetAddressBytes()
                    if ($o -lt $Octet) {
                        $Net += $ba[$o]
                    } ELSEIF ($o -eq $octet) {
                        $factor = 8 + $Octet * 8 - $bit
                        $Divider = [math]::pow(2, $factor)
                        $value = $divider * [math]::Truncate($ba[$o] / $divider)
                        $Net += $value
                    } ELSE {
                        $Net += 0
                    }
                } #Next

                #Format network in CIDR notation
                $NetWork = [string]::join('.', $net) + "/$bit"

                # Try to find this Network in AD Subnets list
                # Most "possible" subnets will not be found, but one should.
                Write-Verbose "Trying : $network"
                try{
                    $de = New-Object System.DirectoryServices.DirectoryEntry("LDAP://" + $DC + "rootDSE")
                    $Root = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$DC$($de.configurationNamingContext)")
                    $ds = New-Object System.Directoryservices.DirectorySearcher($root)
                    $ds.filter = "(CN=$NetWork)"
                    $Result = $ds.findone()
                }catch{
                    $Result = $null
                }

                if ($Result) {
                    write-verbose "AD Site found for $IP"

                    # Try to split out AD Site from LDAP path
                    $Match = "$($Result.GetDirectoryEntry().siteObject)" -match "^CN=(.*),CN=Sites,CN=Configuration.*$"
                    if ($Match) {
                        $ADSite = $Matches[1]
                    } else {
                        $ADSite = "$($Result.GetDirectoryEntry().siteObject)"
                    }

                    $MatchedSubnets += New-Object -TypeName PSObject -Property @{
                        ComputerName = $ComputerName
                        IPAddress    = $IP.ToString()
                        ADSubnetName = $($Result.properties.name).ToString()
                        ADSubnetDesc = "$($Result.properties.description)"
                        ADSite       = $ADSite
                    }
                    $bFound = $true
                }#endif
            }#next

        }#endif
        if ($bFound) {

            if ($AllMatches) {
                # output all the matched subnets

                $MatchedSubnets

            } else {

                # Only output the subnet with the largest mask bits
                [Int32]$MaskBits = 0 # initial value

                Foreach ($MatchedSubnet in $MatchedSubnets) {

                    if ($MatchedSubnet.ADSubnetName -match "\/(?<Bits>\d+)$") {
                        [Int32]$ThisMaskBits = $Matches['Bits']
                        Write-Verbose "ThisMaskBits = '$ThisMaskBits'"

                        if ($ThisMaskBits -gt $MaskBits) {
                            # This is a more specific subnet

                            $OutputSubnet = $MatchedSubnet
                            $MaskBits = $ThisMaskBits

                        } else {
                            Write-Verbose "No match"
                        }
                    } else {
                        Write-Verbose "No match"
                    }
                }
                $OutputSubnet
            }

        } else {

            Write-Verbose "AD Subnet not found for $IP"
            if ($IP -eq $null) {$IP = ""} # required to prevent exception on ToString() below

            New-Object -TypeName PSObject -Property @{
                ComputerName = $ComputerName
                IPAddress    = $IP.ToString()
                ADSubnetName = "Not found"
                ADSubnetDesc = ""
                ADSite       = ""
            }
        }#end if


    }#process

    End {}
}

```
