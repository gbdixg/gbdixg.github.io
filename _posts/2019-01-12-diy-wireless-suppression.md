---
id: 16
title: DIY Wireless Suppression with WMI and PowerShell
date: 2019-01-12T19:15:04+00:00
author: Geoff
description: A Windows 10 WMI permanent event subscription to disable / enable WIFI when Ethernet is connected.
layout: post
permalink: /2019/01/12/diy-wireless-suppression
image: assets/images/thumb-diy-wireless-suppression.png
comments: true
featured: false
hidden: false
categories: [ powershell, wifi, Level300, WMI, Windows10 ]
tags:
  - powershell
  - codesample
  - WMI
  - WIFI
  - Level300
  - Windows10
---

>Wireless Suppression prevents a WIFI connection being used when Ethernet is available.

When you drop a laptop into a docking station with an Ethernet cable, you expect it to use the wired connection instead of than WIFI. This may not be the case on Windows unless wireless is supressed.

This post will cover the problem, common 3rd-party solutions and a built-in solution using WMI and PowerShell

#### Why is this a problem?

* Corporate WIFI can suffer from slow-down due to contention or distance to the Access Point. An available Ethernet connection will provide a more reliable speed.
* Some corporate WIFI networks are not routed to the Intranet and require a VPN connection. WIFI should disconnected when corprate Ethernet is connected to avoid routing failures.
* DHCP address pools can be depleted if all the office clients are consuming two IP addresses unexpectedly.
* I.T. Security teams often want guaranteea that a laptop cannot be connected to public WIFI at the same time as corporate Ethernet (even though routing does not occur by default and Windows Firewall applies per-connection profiles)

#### How does Windows work with multiple active network connections?

Windows uses interface metrics to prioritise traffic when there is more than one active connection. The connection with the lowest metric has the highest priority and should be used.

[Automatic metric assignment has been updated in Windows 10](https://support.microsoft.com/en-us/help/299540/an-explanation-of-the-automatic-metric-feature-for-ipv4-routes) to make it more reliable when dealing with high-speed WIFI.
Prior to Windows 10, it was common for WIFI to have a lower interface metric than fast ethernet.

Use *NETSH* to view the interface metric:

```cmd
netsh interface ipv4 show config
```

>Despite this expected behaviour, removing or disabling the WIFI connection when Ethernet is connected provides a higher level of confidence that network traffic is using the desired connection and addresses some of the other problems described above.

#### When are multiple active connections possible?

In an office, laptops may be set to automatically connect to corporate WIFI, but also use docking stations with Ethernet connections. When users roam to meeting rooms and then return to the docking station, the WIFI connection remains active.
There are obviously other scenarios, but this is a common one.

#### 3rd Party Wireless Suppression

Corporate laptop models from HP and [Dell](https://www.dell.com/support/article/uk/en/ukdhs1/sln285316/how-to-disable-wireless-when-connected-via-wired-connection-on-latitude-and-precision-mobile-workstations?lang=en) include wireless suppression in the BIOS options. The down-side of this method are (a) BIOS settings are not the easiest to manage and (b) hardware models from other vendors may not support wireless suppression.

There are some licensed software solutions such as [Wireless Autoswitch XPV](https://www.wirelessautoswitch.com/), but these obviously incur a cost.

VPN clients, such as [Pulse Secure](https://www.pulsesecure.net/), have an option to implement wireless suppression. This can be a good method, but does tie together two unrelated solutions, making it harder to change the VPN client in the future.

The above solutions work by disabling the Wireless radio when an Ethernet connection is detected and re-enabling it when Ethernet is dropped.

### Custom solution using WMI and PowerShell

[WMI Events](https://docs.microsoft.com/en-us/windows/desktop/wmisdk/receiving-a-wmi-event) allow software and scripts to respond to changes in the state of a Windows device. Permanent subscriptions persist across reboots. To create a WMI event permanent subscription, you need to create the following (administrative rights required):

* An Event Filter
* An Event Consumer
* A Filter to Consumer Binding

The event subscription is then saved in the WMI repository. You can list any existing permanent event subscriptions as follows:

```powershell
$Namespace = 'root\subscription'
Get-WMIObject -Namespace $Namespace -Class __EventFilter
Get-WMIObject -Namespace $Namespace -Class __EventConsumer
Get-WMIObject -Namespace $Namespace -Class __FilterToConsumerBinding
```

#### Event Filter

The filter defines the WMI class to 'watch' for events, how often to monitor for new events and a query that limits when events are 'fired'.

The following filter subscribes to events from the CIM_NetworkPort class, watching every 1 second ('WITHIN 1')
When the MediaConnectState of a networkport changes the event will fire, but only if the additional criteria in the filter are also met.

```PowerShell
SELECT * FROM __InstanceModificationEvent WITHIN 1 WHERE TargetInstance ISA 'CIM_NetworkPort' AND (TargetInstance.MediaConnectState <> PreviousInstance.MediaConnectState) AND (PreviousInstance.InterfaceType='6') AND (NOT PreviousInstance.DriverName LIKE '%jnprva%')"
```

The [interfacetype](https://www.iana.org/assignments/ianaiftype-mib/ianaiftype-mib) is a WMI property that limits the subscription to state changes of an Ethernet interface (6), excluding WIFI or iSCSI etc.

The example also shows how to further filter the subscription to exclude a specific Ethernet connections from generating events - in this case using the network driver name to exclude the Pulse Secure virtual adapter 'jnprva'.

#### Event Consumer

The event consumer dictates how the raised events are handled. There are separate WMI consumer classes that can logg events (text and event log), run 'active scripts' (vbscript/jscript), send SMTP messages and run command lines. To run a PowerShell script, we need to use the [CommandLineEventConsumer](https://docs.microsoft.com/en-us/windows/desktop/wmisdk/commandlineeventconsumer) and set the CommandLineTemplate property as follows:

```powershell
CommandLineTemplate = "$($ENV:SystemRoot)\system32\WindowsPowerShell\v1.0\powershell.exe -NoProfile -ExecutionPolicy Bypass -File `"$ScripttoInvoke`""
```

The `$ScriptToInvoke` is the full path to a .ps1 file that will be called every time an event is raised.

#### Filter to Consumer Binding

The filter to consumer binding links the filter and consumer using the __FilterToConsumerBinding WMI class, creating the event subscription. It takes only two input parameters - the output of creating the event filter and the output of creating the event consumer. See working example below that registers an event subscription.

#### The Script to Invoke

When an event is fired, it will run a PowerShell script to suppress or enable WIFI. This script is very simple:
>When the PowerShell script is triggered it queries the active network connections. If Ethernet is active, any WIFI adapters are disabled. If Ethernet is disconnected, any WIFI adapters are enabled.

### Working Code Example

#### Script to register WMI event subscription

The PowerShell script below is run once on a laptop to register the WMI permanent event subscription. Set the `$ScripttoInvoke` variable to your desired path on the local disk (make sure only administrators can modify files in the selected path otherwise you will create an elevation of privilege vulnerability).

```powershell
<#
    .SYNOPSIS
    Register-WMIEventSubscription.ps1
    Registers a WMI permanent event subscription that calls a PowerShell script when Ethernet connectivity changes
#>
[cmdletBinding()]
[OutputType('PSObject')]
param(
    [Parameter(Position=0,ValueFromPipeline=$True,ValueFromPipelineByPropertyName=$True)]
    [string]$Computername=$env:COMPUTERNAME
    ,
    [string]$ScripttoInvoke = "C:\Windows\Invoke-WirelessSuppression.ps1"
    ,
    [string]$EventName = 'OnEthernetChange'
)
PROCESS{
  $Output = New-Object PSObject | Select-Object Computername,FilterResult,ConsumerResult,BindingResult,OperatorMessage
  $Output.Computername = $Computername
  $Output.OperatorMessage = "OK"

  $wmiParams = @{
      Computername = $Computername
      ErrorAction = 'Stop'
      NameSpace = 'root\subscription'
  }

  if(Test-Connection -ComputerName $Computername -Count 1 -Quiet){

    #REGION -----FILTER-----
    Write-Verbose "Creating Event Filter"
    $wmiParams.Class = '__EventFilter'

    # Fire the event when there is a change to the Win32_NetworkAdapter.NetConnectionStatus of any Ethernet adapter i.e. connected or disconnected
    # InterfaceType=6 is Ethernet only
    # jnprva is the Pulse Secure Juniper virtual adapter

    $wmiParams.Arguments = @{
        Name           = "$EventName"
        EventNamespace = 'root\StandardCIMV2'
        QueryLanguage  = 'WQL'
        Query          = "SELECT * FROM __InstanceModificationEvent WITHIN 1 WHERE TargetInstance ISA 'CIM_NetworkPort' AND (TargetInstance.MediaConnectState <> PreviousInstance.MediaConnectState) AND (PreviousInstance.InterfaceType='6') AND (NOT PreviousInstance.DriverName LIKE '%jnprva%')"
    }

    try{
      $filterResult = Set-WmiInstance @wmiParams
      $Output.FilterResult = "'$($FilterResult.Name)' filter created successfully"

    }catch{
      $Output.FilterResult = "Failed to create - '$_'"
      $Output.OperatorMessage = "One or more errors"

    }
    #ENDREGION

    #REGION -----CONSUMER-----
    # When the event fires, run a PowerShell script
    Write-Verbose "Creating Event Consumer"
    $wmiParams.Class = 'CommandLineEventConsumer'
    $wmiParams.Arguments = @{
      Name = "$EventName"
      CommandLineTemplate = "$($ENV:SystemRoot)\system32\WindowsPowerShell\v1.0\powershell.exe -NoProfile -ExecutionPolicy Bypass -File `"$ScripttoInvoke`""
    }

    try{
      $consumerResult = Set-WmiInstance @wmiParams
      $Output.ConsumerResult = "'$($consumerResult.Name)' consumer created successfully"

    }catch{
      $Output.ConsumerResult = "Failed to create - '$_'"
      $Output.OperatorMessage = "One or more errors"
    }

    #ENDREGION

    #REGION ----- Binding -----
    Write-Verbose "Creating Filter to Consumer Binding"
    $wmiParams.Class = '__FilterToConsumerBinding'
    $wmiParams.Arguments = @{
      Filter = $filterResult
      Consumer = $consumerResult
    }

    try{
      $bindingResult = Set-WmiInstance @wmiParams
      $Output.BindingResult = "Filter to consumer binding created successfully"

    }catch{
      $Output.BindingResult = "Failed to create - '$_'"
      $Output.OperatorMessage = "One or more errors"
    }

  }else{
    $Output.OperatorMessage = "Ping failed. No changes made"
  }
  #ENDREGION

  $Output
}
```

#### Script to disable / enable WIFI

The path and filename of the PowerShell script below must match the `$ScripttoInvoke` variable in the WMI subscription registration script above..
This script is invoked whenever a WMI event fires from the subscription above. It disables and enables WIFI based on the connection state of the Ethernet adapter.

```powershell
<#
    .SYNOPSIS
      Invoke-WirelessSuppression.ps1
      Disables WIFI adapter when Ethernet is connected. Enables WIFI when Ethernet is disconnected
    .DESCRIPTION
      - If WIFI is connected when Ethernet is connected, the WIFI adapter is disabled
      - When an Ethernet connection is lost, if there is a disabled WIFI adapter, it is enabled

      Requires administrative or SYSTEM permissions to disable and enable net adapters
      Designed to be run from a WMI event trigger in the background
#>
[cmdletBinding()]
param()
BEGIN{
  #REGION ----- Support Functions -----

  Function Test-NetAdapterState{
  <#
    .SYNOPSIS
        Determines connection state of Ethernet and WIFI adapters
  #>
  [cmdletBinding()]
    [OutputType('PSObject')]
    param()
    PROCESS{

        $Output = [PSCustomObject][ordered] @{
          Time = Get-Date
          WIFIAdapterCount = 0
          EthernetAdapterCount = 0
          WIFIConnected = $false
          EthernetConnected = $false
          Status = 'OK'
        }

        Try{
          # MSFT_Netadapter.InterfaceType:  Other (1),Ethernet (6),Token ring (9),PPP (23),Loopback (24),ATM (37),WIFI (71),Encapsulated Tunnel (131),Firewire (144)
          # MSFT_Netadapter.MediaConnectState : (0) Unknown (1) Connected (2) Disconnected
          # State: Unknown (0), Present (1), Started (2), Disabled (3)
          $WMIParams =@{
            NameSpace='Root\StandardCimv2'
            Class='MSFT_Netadapter'
            Filter="Virtual='False' AND (InterfaceType=71 OR InterfaceType=6)"
            Property='InterfaceType','InterfaceDescription','Name','MediaConnectState','State'
          }

          # Is there an Ethernet and WIFI adapter?
          $PhysicalAdapters = Get-WmiObject @WMIParams

          if(-not($null -eq $PhysicalAdapters)){

            # Is WIFI connected?
            $WIFIAdapters = @($PhysicalAdapters | Where-Object{$_.InterfaceType -eq 71})
            if(-not($null -eq $WIFIAdapters)){
              $Output.WIFIAdapterCount = $WIFIAdapters.Count

              Foreach($WIFIAdapter in $WIFIAdapters){
                $MediaConnectState = Switch($WIFIAdapter.MediaConnectState){
                  0 {'unknown'}
                  1 {'connected'}
                  2 {'disconnected'}
                }

                if($WIFIAdapter.MediaConnectState -eq 1){
                  $Output.WIFIConnected = $True
                }

              }#foreach

            }else{
              Write-Verbose "WIFI adapter is not connected"
            }

            # Is Ethernet connected?
            $EthernetAdapters = @($PhysicalAdapters | Where-Object{$_.InterfaceType -eq 6})
            if(-not($null -eq $EthernetAdapters)){
              $Output.EthernetAdapterCount = $EthernetAdapters.Count

              Foreach($EthernetAdapter in $EthernetAdapters){
                $MediaConnectState = Switch($EthernetAdapter.MediaConnectState){
                  0 {'unknown'}
                  1 {'connected'}
                  2 {'disconnected'}
                }

                if($EthernetAdapter.MediaConnectState -eq 1){
                  $Output.EthernetConnected = $True
                }
              }#foreach

            }else{
              Write-Verbose "Ethernet is not connected"
            }

          }else{
            Write-Error "No physical adapters found"
            $Output.Status = 'ERROR'
          }

        }catch{
           Write-Error "Error enumerating physical adapters. '$_'"
           $Output.Status = 'ERROR'
        }

        # Return custom object
        $Output

    }#Process
  } #EndFunction


  Function Set-NetAdapterState{
  <#
    .SYNOPSIS
        Enables or disables any adapters of the specified type - Ethernet or WIFI
  #>
    [cmdletBinding()]
    [OutputType('PSObject')]
    param(
      [Parameter()]
      [ValidateSet('WIFI','Ethernet')]
      [string]$AdapterType
      ,
      [Parameter()]
      [ValidateSet('Enable','Disable')]
      [string]$Action
    )
    PROCESS{
        $Source = 'AonWirelessSuppression'

        $AdapterTypeInt = Switch($AdapterType){
          'WIFI' {71}
          'Ethernet' {6}
        }

        if($Action -eq 'Disable'){
          # Is there an enabled adapter?
          # State: Unknown (0), Present (1), Started (2), Disabled (3)
          $WMIParams =@{
            NameSpace='Root\StandardCimv2'
            Class='MSFT_Netadapter'
            Filter="Virtual='False' AND InterfaceType=$AdapterTypeInt AND State=2"
            Property='InterfaceType','InterfaceDescription','Name','MediaConnectState','InterfaceIndex'
          }

          $Adapters = Get-WmiObject @WMIParams

          if(-not($null -eq $Adapters)){
            # Found an enabled adapter - disable it
            $Adapters | ForEach-Object{
                Write-Verbose "Disabling adapter '$($_.InterfaceDescription)' ($($_.Name))..."
                Try{
                   $_ | Disable-NetAdapter -Confirm:$False -ErrorAction Stop # Built-in Windows 10 PS command
                  Write-Verbose "Disabled successfully"

                }catch{
                  Write-Error "Failed to disable. Error '$_'"
                }
            }
          }else{
            Write-Verbose "No enabled $AdapterType adapters were found"
          }

        }elseif($Action -eq 'Enable'){

          # # Is there a disabled adapter?
          # State : (0) Unknown (1) Present (2) Started (3) Disabled
          $WMIParams =@{
            NameSpace='Root\StandardCimv2'
            Class='MSFT_Netadapter'
            Filter="Virtual='False' AND InterfaceType=$AdapterTypeInt AND State=3"
            Property='InterfaceType','InterfaceDescription','Name','MediaConnectState','InterfaceIndex'
          }

          $Adapters = Get-WmiObject @WMIParams

          if(-not($null -eq $Adapters)){
            # Found a disabled adapter - enable it
            $Adapters | ForEach-Object{

              Write-Verbose "Enabling adapter '$($_.InterfaceDescription)' ($($_.Name))..."
              Try{
                $_ | Enable-NetAdapter -Confirm:$False -ErrorAction Stop # Built-in Windows 10 PS command
                Write-Verbose "Enabled successfully"

              }catch{
                Write-Error "Failed to enable. Error '$_'"
              }

            }
          }else{
            Write-Verbose "No disabled $AdapterType adapters were found"
          }

        }#End if

    }#Process
  } #EndFunction

  #ENDREGION

}

PROCESS{

  Write-Verbose "Processing connected status of physical network adapters..."
  $Result = Test-NetAdapterState

  if(-not($Result.Status -eq 'ERROR')){

    if($Result.EthernetConnected -eq $True){

      # Disable WIFI
      Write-Verbose "Ethernet is connected"
      Set-NetAdapterState -AdapterType WIFI -Action Disable

    }elseif(($Result.EthernetConnected -eq $False) -and ($Result.WIFIConnected -eq $False)){

      # Enable WIFI
      Write-Verbose "Ethernet is disconnected and WIFI is disconnected"
      Set-NetAdapterState -AdapterType WIFI -Action Enable

    }else{
      Write-Verbose "No action required"
    }

  }else{

    Write-Error "Error getting connection state. No action taken"
  }

}
```

### Removing a permanent event subscription

If you need to remove the event subscription run the following code:

```powershell
$Computername = $ENV:Computername
$Namespace = 'root\subscription'
$EventName = 'OnEthernetChange'
Get-WMIObject -Computername $Computername -Namespace $Namespace -Class __EventFilter -Filter "Name='$EventName'" | Remove-WmiObject -ErrorAction Stop
Get-WMIObject -Computername $Computername -Namespace $Namespace -Class CommandLineEventConsumer -Filter "Name='$EventName'" | Remove-WmiObject -ErrorAction Stop
Get-WMIObject -Computername $Computername -Namespace $Namespace -Class __FilterToConsumerBinding -Filter "__Path LIKE '%$EventName%'"  | Remove-WmiObject -ErrorAction Stop
```
