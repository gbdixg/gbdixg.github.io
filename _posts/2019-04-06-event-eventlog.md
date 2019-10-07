---
id: 18
title: Export Remote Eventlog
date: 2019-04-06T18:11:01+00:00
author: Geoff
description: Export Windows event logs from remote computers using wevtutil.exe and PowerShell
layout: post
permalink: /2019/04/06/export-eventlog
image: assets/images/thumb-export-eventlog.png
comments: true
featured: false
hidden: false
categories: [ Security, powershell, Level200 ]
tags:
  - powershell
  - Level200
---
> It is often much faster to export an event log on a remote computer and copy the .evtx file over the network.

Although the built-in PowerShell **Get-Winevent** cmdlet can work against remote event logs, it can be painfully slow to retrieve event records when network bandwidth is limited. Copying an .evtx file across the same connection is much faster.

**wevtutil.exe** is a built-in Windows executable that can export event logs on a local or remote computer.  The following PowerShell function uses wevtutil under-the-hood to export event logs and copy them locally.

```powershell
Function Export-EventLog {
	<#
		.SYNOPSIS
			Exports a remote event log to a file.

		.DESCRIPTION
			Uses wevtutil.exe to perform the export on the remote computer
			The log(s) are saved to c:\Windows\Temp and then moved over the network to the local computer
			The resulting log file is $Path\$computername-$logname.evtx
			The file can then be opened in Windows Event Viewer or queried directly using "Get-Winevent -Path...."

			The remote computer must be online and the Windows Firewall must allow inbound RPC and SMB connections

		.PARAMETER  Computername
			The name of the remote computer.

		.PARAMETER  Logname
			The name(s) of the log file to export.

		.PARAMETER  Path
			The local folder path where the output file will be saved
			Default = C:\temp

		.PARAMETER  RemotePath
			The remote folder path used to stage the exported file prior to moving it to the local folder path.
			Environment variables are not supported.
			Default = C:\Windows\Temp

		.EXAMPLE
			PS C:\> Export-EventLog -Computername "PC654321"

			This command will export the System and Application event logs from the remote computer PC654321
			The logs will be exported to c:\windows\temp on the remote computer then moved to
			c:\temp on the local computer

		.EXAMPLE
			PS C:\> Export-EventLog -Computername "PC654321" -LogName "System","Security"

			This command will export the System and Security event logs from the remote computer PC654321

		.EXAMPLE
			PS C:\> "PC654321" | Export-EventLog -LogName "Application","Security"

			This command will export the Application and Security event logs from the remote computer PC654321

		.EXAMPLE
			PS C:\> Get-Winevent -Computername $Computer -Listlog * -EA 0 | Where{$_.RecordCount -gt 0} | Export-EventLog -Computername $Computer

			Thos command will export all the event logs from the remote computer represented by the $computer variable

		.NOTES
			Version: 1.0
	#>
	[CmdletBinding()]
	param(
		[parameter(position = 0, mandatory = $true, valuefromPipeline = $true, valuefrompipelinebypropertyname = $true)]
		[string[]]$Computername = $Env:COMPUTERNAME
		,
		[parameter(position = 1, valuefrompipelinebypropertyname = $true)]
		[string[]]$LogName = @("System", "Application")
		,
		[parameter(position = 2)]
		[ValidateScript( { Test-Path $_ -PathType 'Container' })]
		[string]$Path = "C:\TEMP"
		,
		[parameter(position = 3)]
		[string]$RemotePath = "C:\Windows\Temp"

	)
	PROCESS {

		Foreach ($Name in $Computername) {
			Write-Progress -id 1 -Activity "Computer " -Status "$Name"

			If (Test-Connection -ComputerName $Name -Count 1 -ErrorAction SilentlyContinue) {

				$LogName | ForEach-Object {
					$Log = $_

					$Output = New-Object PSObject | Select Computername, LogName, Path, Result
					$Output.Computername = $Name
					$Output.LogName = $Log

					if ((Get-WinEvent -LogName $Log -ComputerName $Name -MaxEvents 1 -ErrorAction SilentlyContinue | Measure-Object).Count -lt 1) {
						Write-Warning -Message "$Name::$log log is empty. Skipping export"
						return
					}
					$OutputFileName = "$Name-$($Log -replace "/","-").evtx"

					Write-Progress -id 2 -ParentId 1 -Activity "Exporting" -Status "$Log"

					if ($Name -eq $Env:COMPUTERNAME) {
						Write-Verbose "Local computer..."
						$Cmd = "$($Env:windir)\system32\wevtutil.exe epl '$Log' '$Path\$OutputFileName' /r:$Name /ow:True 2>&1"
						$CmdResult = Invoke-Expression -Command $cmd

						if ($CmdResult -eq $Null) {
							Write-Verbose "$Name::$log log export to '$path' = 'Success'"
							$Output.Result = "Success"
						} else {
							Write-Error "$Name::$log log export to '$path' = '$CmdResult'"
							$Output.Result = "Error - $CMDResult"
						}

					} else {
						Write-Verbose "Remote computer..."
						# Wevtutil LogName filepath /r:<remote computer> /ow:<Overwrite true/false>
						$Cmd = "$($Env:windir)\system32\wevtutil.exe epl '$Log' '$RemotePath\$OutputFileName' /r:$Name /ow:True 2>&1"
						$CmdResult = Invoke-Expression -Command $cmd
						if ($CmdResult -eq $Null) {
							# Convert <Drive>:\ to \<Drive>$ for remote connection
							$RemoteUNC = $RemotePath -Replace '(?<Drive>[A-Za-z]+):', '${Drive}$$' # c:\ = c$\

							Write-Verbose "$Name::$log log export to '\\$Name\$RemoteUNC\$OutputFileName' = 'Success'"

							Write-Progress -id 3 -ParentId 1 -Activity "Copying" -Status "$Log"
							Try {
								move-item -path "filesystem::\\$Name\$RemoteUNC\$OutputFileName" -Dest $Path -Force -ErrorAction Stop
								Write-Verbose "$Name::$log log move to '$path' = Success"
								$Output.Result = "Success"
							} Catch {
								Write-Error "$Name::$log log move to '$path' failed - '$_'"
								$Output.Result = "Error - '$_'"
							}

						} else {
							Write-Error "$Name::$log log export to '\\$Name\$RemotePath' = '$CmdResult'"
							$Result = "Error - $CMDResult"
						}


					}#end if

					$Output.Path = "$Path\$OutputFileName"

					$Output

				}#foreach logname

			} else {
				Write-Warning -Message "$Name :: ping failed"
			}

		} #foreach Name

	}#process

}#EndFunction

```
