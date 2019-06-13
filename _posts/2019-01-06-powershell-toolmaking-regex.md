---
id: 15
title: PowerShell Toolmaking via RegEx
date: 2019-01-06T19:15:04+00:00
author: Geoff
description: A PowerShell wrapper around NETSH WLAN, demonstrating a useful RegEx technique
layout: post
permalink: /2019/01/06/powershell-toolmaking-regex
image: assets/images/thumb-powershell-toolmaking-regex.png
comments: true
featured: false
hidden: false
categories: [ powershell, regex, wifi, Level200, toolmaking]
tags:
  - toolmaking
  - powershell
  - codesample
  - regex
  - development
  - Level200
---
PowerShell Tools are re-usable functions that can be used stand-alone or in a pipeline.
In this case, a PowerShell function will "wrap around" a built-in executable to output an object rather than text.

The example below demonstrates one of my **most used and easiest to remember Regular Expression techniques** - using the RegEx "not" operator to set the match boundary.
### Example
The `NETSH WLAN` command output is **ideal for parsing using a RegEx** and converting into a PowerShell object.
```
C:\> netsh wlan show interfaces

There is 1 interface on the system:

    Name                   : Wi-Fi
    Description            : Intel(R) Dual Band Wireless-AC 8260
    GUID                   : 42bce393-237c-4bd4-9d5e-18020ba8bb87
    Physical address       : b7:8a:60:a5:f7:d8
    State                  : connected
    SSID                   : MyWiFi
    BSSID                  : 30:d4:2e:50:de:7f
    Network type           : Infrastructure
    Radio type             : 802.11n
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Connection mode        : Profile
    Channel                : 6
    Receive rate (Mbps)    : 115.6
    Transmit rate (Mbps)   : 115.6
    Signal                 : 97%
    Profile                : MyWiFi

    Hosted network status  : Not available
```
To convert the "fields" above into PowerShell object properties we need to "capture" each name and value in a RegEx match as follows:.
```powershell
$Properties = @{}
$result = netsh wlan show interfaces

$Result | Foreach-Object {

    if ($_ -match '^\s+(?<name>[^:]+):\s(?<value>.*)$') {

        $name = $Matches['name'].Trim()
        $val = $Matches['value'].Trim()

        $Properties.Add($name, $val)
    }

}
```
The Foreach-Object loop above processes the NetSH command output line-by-line.<br>
Each line (the $_ variable) is tested for a match against the RegEx expression using the PowerShell -match operator.

>Rather than trying to come up with a RegEx to match all the possible options of the name field, the "not" operator `[^:]+` captures all the characters until the colon.

#### Breaking down the RegEx
![RegEx](/assets/images/powershell-toolmaking-regex1.png)

I prefer to use named captures in my RegEx. `?\<name>` and `?\<value>` are used within the brackets to name the matches allowing them to be referenced as `$matches['name']` and `$matches['value']`.
If you don't use named captures, they would be $matches[1] and $matches[2].

#### Full script
```powershell
Function Get-WLAN {
<#
  .SYNOPSIS
    Gets the properties of WiFI connections
  .DESCRIPTION
    A PowerShell wrapper around NETSH WLAN to convert the output into a PS object
  .INPUTS
    None
  .OUTPUTS
    PSObject
  .EXAMPLE
    Get-WLAN
  .NOTES
    Author: Geoff Dixon
    Website: www.write-verbose.com
    Twitter: @writeverbose
  #>
    [cmdletBinding()]
    param()
    PROCESS {
      $Properties = @{}
      $result = netsh wlan show interfaces
      if ($LASTEXITCODE -eq 0) {
            $Properties.Add('Computername', $ENV:COMPUTERNAME)
            $Result | Foreach-Object {

                <# Example NETSH command output:
                Name                   : Wi-Fi
                Description            : Intel(R) Dual Band Wireless-AC 8260
                State                  : connected
                SSID                   : MyWiFi
                #>
                if ($_ -match '^\s+(?<name>[^:]+):\s(?<value>.*)$') {

                    $name = $Matches['name'].Trim()
                    $val = $Matches['value'].Trim()

                    $Properties.Add($name, $val)
                }

            } #foreach

        if ($Properties.Count -gt 0) {
            [PSCustomObject]$Properties
        } else {
            Write-Warning "Failed to parse NETSH output"
        }

      } else {
          Write-Warning "Error from NETSH - '$($Error[0])'"
      }

    }#process

}
```
For some practice with Regular Expressions, check out [RegEx Golf](https://alf.nu/RegexGolf/) or [Regex Crosswords](https://regexcrossword.com/). There is even a [Regular Expressions day](https://www.bennadel.com/blog/3629-the-12th-annual-regular-expression-day---june-1st-2019.htm).

