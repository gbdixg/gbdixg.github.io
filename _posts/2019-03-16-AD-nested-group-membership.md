---
id: 17
title: Get direct and nested Active Directory group membership
date: 2019-03-16-2T16:31:05+00:00
author: Geoff
description: More than one way to get direct and indirect AD group membership
layout: post
permalink: /2019/03/16/ad-nested-group-membership
image: assets/images/thumb-ad-nested-group-membership.png
comments: true
featured: false
hidden: false
categories: [ AD, Security, powershell, Level200 ]
tags:
  - ad
  - security
  - powershell
  - Level200
---
Active Directory supports multiple levels of group nesting. A user or computer can therefore be a direct member of a group, or an indirect member via a nested group. The Windows *security token* used to authorise a security principlal's access to resources includes all group memberships - direct and nested.

> The *MemberOf* attribute of an Active Directory user or computer object only contains the direct group memberships.

Listing all groups (direct and nested) is useful in access control troubleshooting.
It is also often used in custom Windows solutions that make use of Active Directory groups, to enable support for group nesting.

## Built-in Method

Whoami.exe is built-in to Windows. The */groups* option shows the direct and nested groups of the current user. It can't be used to lookup other AD users:

```powershell
whoami /groups /fo csv | convertfrom-csv | out-gridview
```

## Active Directory PowerShell Module

The Microsoft AD PowerShell module (available with the RSAT tools) does not contain a command to get user groups. It does contain a command to recursively look-up the members of a group to get the direct and indirect members:

```powershell
Get-ADGroupMember -Identity MyGroup -Recursive
```

# Get-AuthorizationGroup

The 'System.DirectoryServices.AccountManagement' class is a .NET framework (Full version) class.
It contains a method called GetAuthorizationGroups that returns direct and indirect groups for
a user account.

> The GetAuthorizationGroup function below is convenient as it works with a username rather than requiring a distinguishedName

GetAuthorizationGroups can be used for the current user account or an alternate user.
The following PowerShell function demonstrates the method:

```powershell
Function Get-AuthorizationGroup {
    <#
      .SYNOPSIS
        Returns all the groups (including nested) that an Active Directory domain user
         is a member of
      .DESCRIPTION
        Uses the .NET class method:
          System.DirectoryServices.AccountManagement.UserPrincipal.GetAuthorizationGroups()
        Returns the name, Description and distinguishedname of each group the user is
         a member of. The domain must be available or the function will return an error.
      .PARAMETER Userid
        The samAccountName of a domain user
      .EXAMPLE
        Get-AuthorizationGroup -userid jsmith

        This command will list the name,description and distinguishedName of groups that
        the domain user with userid jsmith is a member of
      .NOTES
        Author: @Writeverbose
        Version: 1.0
    #>
    [cmdletbinding()]
    param(
        $Userid
    )
    PROCESS {

        Add-Type -AssemblyName System.DirectoryServices.AccountManagement
        try {
            $principalCtx = new-object System.DirectoryServices.AccountManagement.PrincipalContext('Domain')
            $QueryUser = new-object System.DirectoryServices.AccountManagement.UserPrincipal ($principalCtx)
            $QueryUser.SamAccountName = $Userid
            $principalSearcher = new-object System.DirectoryServices.AccountManagement.PrincipalSearcher
            $principalSearcher.QueryFilter = $QueryUser
            $userPrincipal = $principalSearcher.FindOne()
        } catch {
            Write-error "Failed to access domain : check connectivity ($_)"
        }
        if (-not($null -eq $userPrincipal)) {
            $groups = $userPrincipal.GetAuthorizationGroups()
            $groups | Select samAccountName, Description, distinguishedname
        } else {
          Write-Error "User was not found in the domain"
        }
    }

}
```

## Get-TokenGroup - method 1

> TokenGroups is a constructed attribute of Active Directory user and computer objects that contains all their group memberships.

Constructed attributes are generated when queried. In this case, TokenGroups effectively *explodes* the security token so that the list contains the SIDs of direct and indirect groups.

In the PowerShell function below, a .NET method is then used to convert the SIDs to group names: System.Security.Principal.SecurityIdentifier::Translate()

The function requires the distinguishedName of an Active Directory user or computer as input.

```powershell
Function Get-TokenGroup1 {
    <#
      .SYNOPSIS
        Returns all the groups (including nested) that an Active Directory domain user
         is a member of
      .DESCRIPTION
        Uses the TokenGroups calculated attribute to get group memberships
		A connection to a domain controller is required
        - TokenGroups returns a list of ObjectSIDs
        - The ObjectSID of each group is resolved to a name using
          System.Security.Principal.SecurityIdentifier.Translate()
      .PARAMETER distinguishedName
        The DN of an Active Directory user or computer account
        e.g. CN=jsmith,OU=Europe,OU=Accounts,DC=mydomain,DC=com
      .PARAMETER Server
        The name of an Active Directory Domain Controller or AD Domain
        The default is the %USERDNSDOMAIN% environment variable
      .EXAMPLE
        Get-TokenGroup1 -distinguishedName "CN=jsmith,OU=Europe,OU=Accounts,DC=mydomain,DC=com"

        This example will get the direct and indirect (nested) group memberships of the
        jsmith user accvount in the mydomain Active Directory domain.
      .NOTES
        Author: @WriteVerbose
        Version: 1.0
    #>
    [cmdletBinding()]
    param(
        [parameter(mandatory = $true, position = 0, valuefrompipeline = $True, valuefrompipelinebypropertyName = $True)]
        [ValidatePattern("CN=(.*),((OU|DC)=\w*)*")]
        [string]$DistinguishedName
        ,
        [string]$Server = $ENV:USERDNSDOMAIN
    )
    PROCESS {
        try {
            $ADObj = [ADSI]"LDAP://$Server/$DistinguishedName"
            # Calculated property so need to refreshCache...
            $ADObj.psbase.RefreshCache("tokenGroups")
            $SIDs = $ADObj.psbase.Properties.item("tokenGroups")
        } catch {
            Write-Error "Failed to find user on server '$Server' ($_)"
        }

        if (-not($null -eq $SIDs)) {

            ForEach ($Entry In $SIDs) {
                $SID = New-Object System.Security.Principal.SecurityIdentifier $Entry, 0
                try {
                    $Group = $SID.Translate([System.Security.Principal.NTAccount])
                } catch {
                    Write-warning "Failed to translate SID '$($SID.Value)' to a name"
                    $Group = $null
                }
                [PSCustomObject][ordered]@{
                    UserDN          = $ADObj.Properties.item("DistinguishedName")[0]
                    UserDisplayName = $ADObj.Properties.item("DisplayName")[0]
                    GroupSID        = $SID.Value
                    GroupDomain     = $Group.Value.Split('\')[0]
                    GroupName       = $Group.Value.Split('\')[1]
                }
            }
        } else {
            Write-Warning "Didn't find any group memberships for user '$DistinguishedName'"
        }
    }
}

```

## Get-TokenGroup - Method 2

The script below also uses the TokenGroups attribute to obtain the directed and nested group membership.  The only difference from Method1 is the way the group SIDs are resolved to names.

In the example below, each SIDs is converted to an *Octet string* format that can be looked-up in AD using an LDAP query. The entire list of group Octet Strings is concatenated into a single LDAP query that returns the group names.

There is no real benefit to using Method2 over Method1. Method2 was the first way I solved this problem, before becoming aware of the .Net options that are more concise.

```powershell
Function Get-TokenGroup2 {
    [cmdletBinding()]
    <#
	.SYNOPSIS
		Used to get the direct and nested group membership of an AD security principal (user or computer)
	.DESCRIPTION
		Uses the TokenGroups calculated attribute to get group memberships
		A connection to a domain controller is required
		- TokenGroups returns a list of ObjectSIDs
		- Script then performs an AD Query filtered using the SIDs to get the displayNames

		Returns a string array of group names (without domain portion)
	.PARAMETER  DistinguishedName
		The distinguishedName of an Active Directory user or computer
		e.g. "CN=jsmith,OU=Europe,OU=Accounts,DC=mydomain,DC=com"
	.EXAMPLE
        PS C:\> Get-TokenGroup2 "CN=jsmith,OU=Europe,OU=Accounts,DC=mydomain,DC=com"

        This command will get the direct and indirect (nested) group membersjips of
        the jsmith user account in the mydomain Active Directory domain
    .NOTES
        Author:     @WriteVerbose
		Version:    1.0
	.Link
		http://msdn.microsoft.com/en-us/library/windows/desktop/ms680275(v=vs.85).aspx
#>
    param(
        [parameter(mandatory = $true, position = 0, valuefrompipeline = $True, valuefrompipelinebypropertyName = $True)]
        [ValidatePattern("CN=(.*),((OU|DC)=\w*)*")]
        [string]$DistinguishedName
    )
    BEGIN {
        #REGION --- Support Functions ---
        Function Convert-ToLDAPFilter($Octet) {
            # This function is used to build an LDAP search string to convert Group SIDs from TokenGroups to samAccountNames

            [System.Text.StringBuilder]$sb = New-Object System.Text.StringBuilder("") # output object

            # Input is either a Byte or a Byte array
            if ($Octet.GetType().BaseType.Name -eq "Byte") {
                # Single entry
                $HexString = "\$(([System.BitConverter]::ToString($Octet)).Replace('-','\'))"
                $sb.Append("(objectSid=$HexString)") | Out-Null

            } elseif ($Octet.GetType().BaseType.Name -eq "Array") {
                # multiple entries
                # convert octet byte array to hex string and build ldap filter
                $sb.Append("(|") | Out-Null

                foreach ($group in $Octet) {
                    $HexString = "\$(([System.BitConverter]::ToString($group)).Replace('-','\'))"
                    $sb.Append("(objectSid=$HexString)") | Out-Null
                }

                $sb.Append(")") | Out-Null
            } else {
                # no entries
            }
            $sb.ToString() #return
        }

        # This function is used to execute the LDAP search string, to convert Group SIDs to samAccountNames
        Function Find-DirectoryEntry {
            <#
		.Synopsis
		    Finds an Active Directory object using a directory search
		.Description
		    Helper function to search AD using an LDAP query
		.Parameter Filter
			The filter portion of the LDAP query
		.Parameter Root
			The root container for the query
		.Parameter Property
			An array of parameters to be returned
		.Parameter Scope
			The depth of the query - base,onelevel or subtree
		 .Example
		 	PS C:\>Find-DirectoryEntry -Filter "(objectClass=computer)" -Property "*"

			This command will return all computer objects in the domain of the current user and include all available properties
		 .Example
		 	PS C:\>Find-DirectoryEntry -Filter "(objectClass=user)" -Property "displayName,givenName,sn" -root "cn=user,dc=my,dc=home"
			This command will return all user objects in the Users container of the my.home domain and include the displayName, givenName and surname properties
		 .Notes
			Version:        1.0
			Author:         @WriteVerbose
		#>
            param(
                [parameter(position = 0, mandatory = $false)]
                [string]$Filter = "*"
                ,
                [parameter(position = 1, mandatory = $false)]
                [string]$Root = $env:USERDNSDOMAIN
                ,
                [parameter(position = 2, mandatory = $false)]
                [string[]]$Property = "*"
                ,
                [parameter(position = 3, mandatory = $false)]
                [string]$Scope = "subtree"
            )
            try {
                $DE = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$Root")
                $Searcher = New-Object System.DirectoryServices.DirectorySearcher
                $Searcher.SearchRoot = $DE
                $Searcher.PageSize = 1000
                $Searcher.Filter = $Filter
                $Searcher.SearchScope = $Scope

                foreach ($entry in $Property) { $Searcher.PropertiesToLoad.Add($entry) }
                $Results = $Searcher.FindAll()

                # Return the DirectorySearcher.SearchResult object
                return $Results
            } catch [Exception] {

                Write-Error "Failed to find directory entry: $($_.Exception.MessageDetails)"
                Return $null
            }
        }

        #ENDREGION
    }
    PROCESS {

        # Requires a connection to a DC to complete
        try {
            $ADObj = [ADSI]"LDAP://$DistinguishedName"
            $ADObj.GetInfoEx(@("tokengroups"), 0)
            $groups = $ADObj.Get("tokengroups") # this returns an array of group SIDs
        } catch {
            Write-Error "Failed to find user on default domain ($_)"
        }

        if (-not($null -eq $groups)) {

            $Filter = Convert-ToLDAPFilter -Octet $groups # Build an LDAP query to convert the group SIDs to names

            # Execute the query against AD. Returns a SearchResult collection
            $Result = Find-DirectoryEntry -Filter $filter -Property "SamAccountName"

            # Convert the SearchResult collection to a string array of Group Names
            [String[]]$GroupNames = $($Result | Foreach {
                    if ($_ -as [System.DirectoryServices.SearchResult]) {
                        $_.Properties.Item("samAccountName")[0]
                    }
                })

            $GroupNames # output
        } else {
            Write-Warning "Didn't find any group memberships for user '$DistinguishedName'"
        }

    }#process
}

```