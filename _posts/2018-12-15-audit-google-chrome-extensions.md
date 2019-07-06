---
id: 17
title: Audit Chrome Extensions with PowerShell
date: 2018-12-15-2T18:24:34+00:00
author: Geoff
description: A PowerShell script to list the installed Google Chrome extensions on a local or remote computer
layout: post
permalink: /2018/12/15/audit-google-chrome-extensions
image: assets/images/thumb-audit-google-chrome-extensions.png
comments: true
featured: false
hidden: false
categories: [ PowerShell, Browser, Level200, Chrome, Security ]
tags:
  - security
  - PowerShell
  - Level200
  - Chrome
---

The code below is a PowerShell function to get the installed Google Chrome browser extensions from a local or remote Windows computer.

> Chrome Browser Extensions install into the user profile and do not appear in the Add/Remove Programs list.

Chrome Extensions are a challenge to audit due to the way they install and lack of enumeration options.
The PowerShell script below gets the installed extensions using the following method:

* Get the **extension IDs** from the folders names under ```%userprofile%\AppData\Local\Google\Chrome\User Data\Default\Extensions```
* Lookup the **extension name** on the Chrome Web Store using the extension ID
* Get the **extension version** from the ```manifest.json``` file in the extension folder

### Example script output

```cmd
C:\> Get-ChromeExtension | Select Name,Version,Description | ft -AutoSize

Name                            Version      Description
----                            -------      -----------
Docs                            0.10         Create and edit documents
Google Drive                    14.1         Google Drive: create, share and keep all your stuff in one place.
YouTube                         4.2.8        The official YouTube website
Sheets                          1.2          Create and edit spreadsheets
Google Docs Offline             1.4          Get things done offline with the Google Docs family of products.
Google Wallet                   1.0.0.4
Gmail                           8.1          Fast, searchable email with less spam.
Chrome Cast                     6618.312.0.2
Slides                          0.10         Create and edit presentations
Docs                            0.10         Create and edit documents
Google Drive                    14.2         Google Drive: create, share and keep all your stuff in one place.
YouTube                         4.2.8        The official YouTube website
OneTab                          1.18         Save up to 95% memory and reduce tab clutter
uBlock Origin                   1.20.0       Finally, an efficient blocker. Easy on CPU and memory.
Dark Reader                     4.7.12       Dark mode for every website. Take care of your eyes, use dark theme for night and daily browsing.
Share link via email            3.2.1        Adds a button and context menu item to send the page URL or a link URL via email
Sheets                          1.2          Create and edit spreadsheets
Google Docs Offline             1.7          Get things done offline with the Google Docs family of products.
Pinterest Save Button           4.0.82       Save the things you find on the Web.
Google Wallet                   1.0.0.4
ColorPick Eyedropper            0.0.2.29     An eye-dropper &amp; color-picker tool that allows you to select color values from webpages.
Gmail                           8.2          Fast, searchable email with less spam.
Chrome Cast                     7519.422.0.3
```

### Get-ChromeExtension Script

```powershell
function Get-ChromeExtension {
<#
 .SYNOPSIS
    Gets Chrome Extensions from a local or remote computer
 .DESCRIPTION
    Gets the name, version and description of the installed extensions
    Admin rights are required to access other profiles on the local computer or
    any profiles on a remote computer.
    Internet access is required to lookup the extension ID on the Chrome web store
 .PARAMETER Computername
    The name of the computer to connect to
    The default is the local machine
 .PARAMETER Username
    The username to query i.e. the userprofile (c:\users\<username>)
    If this parameter is omitted, all userprofiles are searched
 .EXAMPLE
    PS C:\> Get-ChromeExtension

    This command will get the Chrome extensions from all the user profiles on the local computer
 .EXAMPLE
    PS C:\> Get-ChromeExtension -username Jsmith

    This command will get the Chrome extensions installed under c:\users\jsmith on the local computer

 .EXAMPLE
    PS C:\> Get-ChromeExtension -Computername PC1234,PC4567

    This command will get the Chrome extensions from all the user profiles on the two remote computers specified
 .NOTES
    Version 1.0
#>
    [cmdletbinding()]
    PARAM(
        [parameter(Position = 0)]
        [string]$Computername = $ENV:COMPUTERNAME
        ,
        [parameter(Position = 1)]
        [string]$Username
    )
    BEGIN {

        function Get-ExtensionInfo {
            <#
         .SYNOPSIS
            Get Name and Version of the a Chrome extension
         .PARAMETER Folder
            A directory object (under %userprofile%\AppData\Local\Google\Chrome\User Data\Default\Extensions)
        #>
            [cmdletbinding()]
            PARAM(
                [parameter(Position = 0)]
                [IO.DirectoryInfo]$Folder
            )
            BEGIN{

                $BuiltInExtensions = @{
                    'nmmhkkegccagdldgiimedpiccmgmieda' = 'Google Wallet'
                    'mhjfbmdgcfjbbpaeojofohoefgiehjai' = 'Chrome PDF Viewer'
                    'pkedcjkdefgpdelpbcmbmeomcjbeemfm' = 'Chrome Cast'
                }

            }
            PROCESS {
                # Extension folders are under %userprofile%\AppData\Local\Google\Chrome\User Data\Default\Extensions
                # Folder names match extension ID e.g. blpcfgokakmgnkcojhhkbfbldkacnbeo
                $ExtID = $Folder.Name

                if($Folder.FullName -match '\\Users\\(?<username>[^\\]+)\\'){
                    $Username = $Matches['username']
                }else{
                    $Username = ''
                }

                # There can be more than one version installed. Get the latest one
                $LastestExtVersionInstallFolder = Get-ChildItem -Path $Folder.Fullname | Where-Object { $_.Name -match '^[0-9\._-]+$' } | Sort-Object -Property CreationTime -Descending | Select-Object -First 1 -ExpandProperty Name

                # Get the version from the JSON manifest
                if (Test-Path -Path "$($Folder.Fullname)\$LastestExtVersionInstallFolder\Manifest.json") {

                    $Manifest = Get-Content -Path "$($Folder.Fullname)\$LastestExtVersionInstallFolder\Manifest.json" -Raw | ConvertFrom-Json
                    if ($Manifest) {
                        if (-not([string]::IsNullOrEmpty($Manifest.version))) {
                            $Version = $Manifest.version
                        }
                    }
                } else {
                    # Just use the folder name as the version
                    $Version = $LastestExtVersionInstallFolder.Name
                }

                if($BuiltInExtensions.ContainsKey($ExtID)){
                    # Built-in extensions do not appear in the Chrome Store

                    $Title = $BuiltInExtensions[$ExtID]
                    $Description = ''

                }else{
                    # Lookup the extension in the Store
                    $url = "https://chrome.google.com/webstore/detail/" + $ExtID + "?hl=en-us"

                    try {
                        # You may need to include proxy information
                        # $WebRequest = Invoke-WebRequest -Uri $url -ErrorAction Stop -Proxy 'http://proxy:port' -ProxyUseDefaultCredentials

                        $WebRequest = Invoke-WebRequest -Uri $url -ErrorAction Stop

                        if ($WebRequest.StatusCode -eq 200) {

                            # Get the HTML Page Title but remove ' - Chrome Web Store'
                            if (-not([string]::IsNullOrEmpty($WebRequest.ParsedHtml.title))) {

                                $ExtTitle = $WebRequest.ParsedHtml.title
                                if ($ExtTitle -match '\s-\s.*$') {
                                    $Title = $ExtTitle -replace '\s-\s.*$',''
                                    $extType = 'ChromeStore'

                                } else {
                                    $Title = $ExtTitle
                                }
                            }

                            # Screen scrape the Description meta-data
                            $Description = $webRequest.AllElements.InnerHTML | Where-Object { $_ -match '<meta name="Description" content="([^"]+)">' } | Select-object -First 1 | ForEach-Object { $Matches[1] }
                        }
                    } catch {
                        Write-Warning "Error during webstore lookup for '$ExtID' - '$_'"

                    }
                }

                [PSCustomObject][Ordered]@{
                    Name        = $Title
                    Version     = $Version
                    Description = $Description
                    Username    = $Username
                    ID          = $ExtID
                }

            }
        }

        $ExtensionFolderPath = 'AppData\Local\Google\Chrome\User Data\Default\Extensions'

    }

    PROCESS {
        Foreach ($Computer in $Computername) {

            if ($Username) {
                # Single userprofile
                $Path = Join-path -path "fileSystem::\\$Computer\C$\Users\$Username" -ChildPath $ExtensionFolderPath
                $Extensions = Get-ChildItem -Path $Path -Directory -ErrorAction SilentlyContinue

            } else {
                # All user profiles that contain this a Chrome extensions folder
                $Path = Join-path -path "fileSystem::\\$Computer\C$\Users\*" -ChildPath $ExtensionFolderPath
                $Extensions =@()
                Get-Item -Path $Path -ErrorAction SilentlyContinue | ForEach-Object{

                    $Extensions += Get-ChildItem -Path $_ -Directory -ErrorAction SilentlyContinue
                }

            }

            if (-not($null -eq $Extensions)) {

                Foreach ($Extension in $Extensions) {

                    $Output = Get-ExtensionInfo -Folder $Extension
                    $Output | Add-Member -MemberType NoteProperty -Name 'Computername' -Value $Computer
                    $Output

                }

            } else {
                Write-Warning "$Computer : no extensions were found"

            }

        }#foreach
    }
}

```
