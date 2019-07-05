---
id: 11
title: PSPam - Privileged Access Management via PowerShell
date: 2018-11-24T16:40:11+00:00
author: Geoff
description: Use PowerShell and JEA to create a local Privilged Access Management solution
layout: post
permalink: /2018/11/24/PSPam-privileged-access-management
image: assets/images/thumb-PSPam-privileged-access-management.png
comments: true
featured: false
hidden: false
categories: [ powershell, security, Level300, JEA, Windows10 ]
tags:
  - powershell
  - codesample
  - JEA
  - Security
  - Level300
  - Windows10
---

```Update 2019-01-12: ``` [A Microsoft security update](https://devblogs.microsoft.com/powershell/windows-security-change-affecting-powershell/) appears to have broken this solution.

## Background

Most users logging on to a Windows client fall into one of the following categories:

* Full administrator
* Standard user

Sometimes there is a need for a *standard user* to perform an elevated task, but granting full administrative access is undesirable.

This article will describe a built-in solution for secure delegation of elevated tasks on Windows client computers, without needing to grant full administrative access.

>***Privileged Access Management* (PAM) works at a granualar level to elevate privileges for specific tasks run by authorised users**


Enterprise PAM solutions usually involve an agent service runing on the endpoint that delegates elevated access using a rule-based approach. Most provide a basic framework and it is up to the customer to determine the rules that allow or deny elevation. Additional features include centralised logging of elevation attempts and customisable messages to advise users of their responsibilities or restrictions.

A simpler PAM solution can be created using built-in features of Windows 10...

## PowerShell and JEA

[Just Enough Administration](https://docs.microsoft.com/en-us/powershell/jea/overview) is a PowerShell solution used to delegate  remote administration of Windows Sevrers. It uses a built-in feature called *Constrained Runspaces* to limit the commands that can be run in a PowerShell session. Role Based Access Control allows users to have different command and language capabilities within a contrained runspace session, depending on their group membership.

>Although JEA is normally used in PowerShell remoting, it can also be used in *local loop-back mode*.

Local loop-back configures a PowerShell constrained endpoint so it is only accessible from an interactive session on the computer. Over-the-network connections to the endpoint are not possible. This option is well-suited to the PSPAM solution...

## PSPam Solution components

The *PSPam* solution, built on PowerShell and JEA, consists of the following text file components:

* *PowerShell Session Configuration file* (.pssc)
* JEA *Role Capabilities* file (.psrc)
* PowerShell module (.psm1) containing allowed functions

### PowerShell Session Configuration file

By default, there are 3 built-in PowerShell endpoints on Windows 10 - *microsoft.powerShell*, *microsoft.powershell32* and *microsoft.powershell.workflow*.

A Session Configuration file is used to register a new PowerShell endpoint with custom properties.
In the 'PAM.pssc' example below, a PowerShell session that uses this configuration will be restricted to *no language mode*, but it will run in the security context of a temporary local administrator account. This means any commands run in the session are evelvated to administrator.

In addition, members of the 'Domain Users' group that connect will have the *StdUser.psrc* role capabilities settings applied (see further below).

```powershell
# PAM.pssc - PowerShell session configuration file
@{
    # Version number of the schema used for this document
    SchemaVersion = '2.0.0.0'

    # ID used to uniquely identify this document
    GUID = '675a637c-fe79-4e02-b9a5-d7c3fbb10583'

    # Author of this document
    Author = 'Wingtip Toys'

    # Description of the functionality provided by these settings
    Description = 'Creates an endpoint for Privileged Access Management'

    # RestrictedRemoteServer = 'No Language Mode', preventing use of C# and Win32 calls
    # There are only 8 supported commands by default: Clear-Host, Get-Command, Get-Help, Get-FormatData,
    #   Select-Object, Out-Default, Measure-Object, Exit-PSSession
    SessionType = 'RestrictedRemoteServer'

    # Directory to place session transcripts for this session configuration
    TranscriptDirectory = 'C:\ProgramData\PAM\Transcripts'

    # Whether to run this session configuration as the machine's (virtual) administrator account
    RunAsVirtualAccount = $true

    # User roles (security groups), and the role capabilities that should be applied to them when applied to a session
    RoleDefinitions = @{
        'WingtipDomain\Domain Users' = @{
            'RoleCapabilities' = 'StdUser'
        }
    }
}
```

Create the new PowerShell endpoint using the configuration file as follows:

```powershell
Register-PSSessionConfiguration -Name PAM -Path .\PAM.pssc
```

> TIP: You can see the full list of possible options for a configuration file using ```New-PSSessionConfigurationFile -Full -Path .\Example.pssc```

Update the security description on the new PowerShell endpoint so that network connections are denied and it can only be used in *local loop-back* mode:

```powershell
Set-PSSessionConfiguration -Name PAM -SecurityDescriptorSddl 'O:NSG:BAD:P(D;;GA;;;NU)(A;;GA;;;DU)S:P(AU;FA;GA;;;WD)(AU;SA;GXGW;;;WD)'
```

The Security Descriptor on the PowerShell endpoint should now allow 'Domain Users' Full Control and Deny 'Builtin\NETWORK'

![Security Descriptor](/assets/images/PSPam-privileged-access-management1.png){:class="img-responsive"}


If you want to construct your own SDDL string, the UI can be used after the endpoint has been registered:

* Run ```Set-PSSessionConfiguration -Name PAM -ShowSecurityDescriptorUI```
* Edit the required permissions in the UI and click *OK*
* Run ```Get-PSSessionConfiguration -Name PAM | Select SecurityDescriptorSddl```

At this stage we have a registered a custom PowerShell endpoint that can only be accessed from a local interactive or remote desktop logon, but it is barely functional. The next step will enable the role capabilities that provide the granular control of elevated tasks.

### JEA Role Capabilities file

The Role Capabilities file specified for 'Domain Users' in the Session Configuration file is used to define which commands can be run in the PowerShell session.

Since the PowerShell session runs as a temporary administrator, the allowed commands have elevated access and should be carefully controlled. Luckily, they control is granular, evening restricting commands to specific parameter options.

In the example Role Capabilities file below, Restart-Service is an allowed command, but only when used to restart the W32Time service.

```powershell
# StdUser.psrc - JEA role capabilities file
@{
    # ID used to uniquely identify this document
    GUID             = '9d836aae-f62f-4b29-ad3b-3d404beacb74'

    # Author of this document
    Author           = 'WingTip R&D'

    # Description of the functionality provided by these settings
    Description      = 'Privileged Access Management Role Capabilities file for Standard Users'

    # Company associated with this document
    CompanyName      = 'WingTipToys'

    # Copyright statement for this document
    Copyright        = '(c) 2019 WingTip PLC. All rights reserved.'

    # A custom module that will be auto-loaded
    # Cmdlets and functions in the module still need to be made 'visible'
    ModulesToImport  = 'PAM.psd1'

    # Allowed cdmlets - from the built-in and custom modules on the computer
    # An allowed cmdlet may be limited to specific parameters
    VisibleCmdlets   = @{Name = 'Restart-Service'; Parameters = @{Name = 'Name'; ValidateSet = 'W32Time' } }

    # Allowed functions from the built-in and custom modules on the computer
    VisibleFunctions = 'Restart-NetAdapter', 'Get-NetAdapter', 'TabExpansion2', 'Update-Hosts'
}
```

It is useful to allow the TabExpansion2 function if the custom PowerShell session is used interactively.

> TIP: You can see the full list of possible options for a role capabilities file using ```New-PSRoleCapabilityFile -Path .\Example.psrc```

### PowerShell module containing allowed functions

A PowerShell module is used to load the custom functions within the session that provide the granular admin delegation in your environment. Because the custom PowerShell session runs as the *virtual* administrator account, the functions will run elevated even when run by a standard user.

There are no language restrictions on the commands that are used inside these module functions.

> You need to write the PowerShell functions in this module - PSPAM just provides the framework.

For example, you could create a function that toggles a value in a configuration file located under *c:\program files*. This would normally require full administrative access to the computer.

Perhaps you want to provide a self-service troubleshooting script that needs to read the *Microsoft-Windows-GroupPolicy/Operational* event log. Again this would normally require full administrative access to the Windows 10 computer.

```powershell
# PAM.psm1 - Example Module containing custom commands to run elevated in the PAM session

$PAMConfig = Get-PSSessionConfiguration -Name 'PAM'
If ($null -eq $PAMConfig) {
    # Need to register the PowerShell Endpoint

    # Register using the session configuration file
    Register-PSSessionConfiguration -Name 'PAM' -path "$PSScriptRoot\PAM.pssc" | out-null

    # Update the security descriptor to remove over-the-network access
    Set-PSSessionConfiguration -Name 'PAM' -SecurityDescriptorSddl 'O:NSG:BAD:P(D;;GA;;;NU)(A;;GA;;;DU)S:P(AU;FA;GA;;;WD)(AU;SA;GXGW;;;WD)' | out-null
}

# Define the custom functions that can run elevated

Function Update-Hosts {
    <#
        .SYNOPSIS
          Example - Updating the HOSTS file requires elevated access
    #>
    [cmdletBinding()]
    param(
        [string]$IP
        ,
        [string]$Hostname
    )
    PROCESS {
        $Text = ("{0}     {1}" -f $IP, $Hostname)
        try {
            $Text | Add-Content -Path 'C:\windows\System32\drivers\etc\hosts' -ErrorAction Stop
        } catch {
            throw "Failed to update hosts '$_'"
        }
    }
}

Function Get-GPEvent {
    <#
        .SYNOPSIS
          Example - Reading the GroupPolicy event log requires elevated access
    #>
    [cmdletBinding()]
    param()
    PROCESS {
        Get-WinEvent -LogName 'Microsoft-Windows-GroupPolicy/Operational'
    }
}
```

### Where to save the files?

When a PowerShell PAM session is started, it searches for a role capabilities file matching the name in the Session Configuration file, with a .psrc extension - i.e. ```StdUser.psrc```

The ```StdUser.psrc``` file must be located in a module in the ```$ENV:PSModulePath```, in a subfolder called 'RoleCapabilities'.

We can combine all three components to create a module that contains the Session Configuration file and the Role Capabilities File along with the custom functions that delegate admin tasks to standard users.

The module folder structure would be as follows:

```cmd
C:\Program Files\WindowsPowerShell\Modules
            \PAM
             - pam.psd1
             - pam.psm1
             - pam.pssc
                \RoleCapabilities
                 - StdUser.psrc
```

### Using the PAM PowerShell Session

There a numerous options for how the custom PowerShell session is used, for example:

* A Start Menu shortcut that launches an interactive PowerShell PAM prompt
* A PowerShell or C# GUI running as a standard user that calls individual elevated functions in the PAM session
* A Menu System that offers users a list of available elevated PAM actions

A shortcut would use the following command line:

```cmd
PowerShell.exe -ConfigurationName PAM
```

A GUI would connect to a PAM session and then invoke commands in the session as follows:

```powershell
# PowerShell GUI running as a standard user would run:
$session = New-PSSession -ComputerName localhost -ConfigurationName PAM -EnableNetworkAccess
Invoke-Command -Session $session -ScriptBlock {Update-ConfigFile -Environment "DEV"}
#...
# then clean-up when finished
Remove-PSSession -Session $session
```

> TIP: The ```-EnableNetworkAccess``` switch above is essential to avoid an 'access denied' error

### Download

[PAM-ModuleExample.zip](/assets/files/PAM-ModuleExample.zip)
