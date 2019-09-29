---
id: 17
title: Active Directory System Information
date: 2019-03-30T18:45:05+00:00
author: Geoff
description: Get Active Directory info via ADSystemInfo COM object
layout: post
permalink: /2019/03/30/adsysteminfo
image: assets/images/thumb-adsysteminfo.png
comments: true
featured: false
hidden: false
categories: [ AD, Security, powershell, Level200 ]
tags:
  - ad
  - powershell
  - Level200
---
> ADSystemInfo is a built-in COM object in Windows 7, 8.1 and 10 that simplifies lookup of Active Directory user and computer information.

ADSystemInfo can only return information about the local computer and current user. A domain controller must be accessible when the functions are called.

Its simple to instantiate COM objects in PowerShell. The function below shows how to use this object.

```powershell
Function Get-ADSystemInfo{
<#
	.Synopsis
		Used to lookup specific AD user/computer object properties of the current session
	.Description
		Uses "ADSystemInfo" COM object to get Active Directory attributes for the current user and computer

	.Example
		PS C:\>Get-ADSystemInfo

		ComputerDN      : CN=EGBLHCNU335BQCG,OU=GBR,OU=Workstations,OU=EU,OU=Regions,DC=mycompany,DC=com
		SiteName        : EULON
		DomainDNSName   : mycompany.com
		DomainShortName : MYCOMPANY
		ForestDNSName   : mycompany.com
		IsNativeMode    : True
		PDCRoleOwner    : CN=527616-NAADCP01,CN=Servers,CN=Global,CN=Sites,CN=Configuration,DC=mycompany,DC=com
		SchemaRoleOwner : CN=527616-NAADCP01,CN=Servers,CN=Global,CN=Sites,CN=Configuration,DC=mycompany,DC=com
		UserDN          : CN=gdixon2,OU=Users,OU=GBR,OU=Accounts,OU=EU,OU=Regions,DC=mycompany,DC=com

	.Notes
		Version:        1.0

	.Link
		http://msdn.microsoft.com/en-us/library/aa705962(VS.85).aspx
#>
[CmdletBinding()]
Param()
	Process{

		$Output = New-Object -TypeName PSObject |
				Select ComputerDN,SiteName,DomainDNSName,DomainShortName,ForestDNSName,IsNativeMode,PDCRoleOwner,SchemaRoleOwner,UserDN


			$obj = new-object -com ADSystemInfo
			$type = $obj.gettype()

			$Output.ComputerDN = $type.InvokeMember("ComputerName","GetProperty",$null,$obj,$null)
			$Output.SiteName = $type.InvokeMember("sitename","GetProperty",$null,$obj,$null)
			$Output.DomainDNSName = $type.InvokeMember("DomainDNSName","GetProperty",$null,$obj,$null)
			$Output.DomainShortName = $type.InvokeMember("DomainShortName","GetProperty",$null,$obj,$null)
			$Output.ForestDNSName = $type.InvokeMember("ForestDNSName","GetProperty",$null,$obj,$null)
			$Output.IsNativeMode = $type.InvokeMember("IsNativeMode","GetProperty",$null,$obj,$null)
			$Output.PDCRoleOwner = ($type.InvokeMember("PDCRoleOwner","GetProperty",$null,$obj,$null) -replace "CN=NTDS Settings,","")
			$Output.SchemaRoleOwner = ($type.InvokeMember("SchemaRoleOwner","GetProperty",$null,$obj,$null) -replace "CN=NTDS Settings,","")
			$Output.UserDN = $type.InvokeMember("UserName","GetProperty",$null,$obj,$null)

			$Output
	}
}
```


