---
layout: post
title: Content Location Requests
date: 2016-02-21
tags: ["ConfigMgr","Powershell","SCCM"]
---

So things have been busy recently but I would like to share something that I have found to be helpful. I have been troubleshooting instances of installs via SCCM failing due to content not being available to download. However when checking the status of the content it reports that is was successfully distributed to the distribution point. There is a long story about why that I will share later. But in the process of trying to proactively find remote locations and packages that would experience this issue I needed a way to mimic the content location request of the client. The post by [Noah Swanson](https://ndswanson.wordpress.com/2015/12/07/configuration-manager-content-location-requests/) was very helpful. For reference here is Noah's powershell :
```powershell
[void] [Reflection.Assembly]::LoadWithPartialName("System.Diagnostics")
[void] [Reflection.Assembly]::LoadFile("c:\Microsoft.ConfigurationManagement.Messaging.dll")
# Set up the objects
$httpSender = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Sender.Http.HttpSender

$clr = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ConfigMgrContentLocationRequest

$cmReply = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ContentLocationReply

# Discover (the try/catch is in case the ADSI COM
# object cannot be loaded (WinPE).
try
{
   $clr.Discover()
}
catch
{
   $clr.LocationRequest.ContentLocationInfo.IPAddresses.DiscoverIPAddresses() $clr.LocationRequest.ContentLocationInfo.ADSite.DiscoverADSite()
   $clr.LocationRequest.Domain = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ContentLocationDomain
   $clr.LocationRequest.Domain.Name = $myDomain
   $clr.LocationRequest.Forest = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ContentLocationForest
   $clr.LocationRequest.Forest.Name = $myForest
}

# Define our SCCM Settings for the message
$clr.SiteCode = $ManagementPointSiteCode
$clr.Settings.HostName = $ManagementPoint
$clr.LocationRequest.Package.PackageId = $PackageID
$clr.LocationRequest.Package.Version = $PackageVersion

# If we want to "spoof" a request from a different
# AD Site, we can just pass that AD Site in here.
if ($myCustomAdSite -ne $null)
{
   $clr.LocationRequest.ContentLocationInfo.ADSite = $myCustomAdSite
}

# Validate
$clr.Validate([Microsoft.ConfigurationManagement.Messaging.Framework.IMessageSender]$httpSender)

# Send the message
$cmReply = $clr.SendMessage($httpSender)

# Get response
$response = $cmReply.Body.Payload.ToString()

while($response[$response.Length -1] -ne '>')
{
   $response = $response.TrimEnd($response[$response.Length -1])
}

 Write-Output $response
```
First run through I update the path to the location of the Microsoft.ConfigurationManagement.Messaging.dll on my system and set the site code, Management Point, package id, and package version to valid values for my environment. Everything works, kudos to Noah.  Next I set the AD Site to a value for one of the remote sites. Once again everything works but the response has distribution points for my local client's ip and the remote AD Site.  So it looks as though this code was developed to run on a local client. I needed to check numerous packages at numerous locations.  A little experimentation and I found that you do not need all of the information populated by the discover method. Just setting the AD Site was sufficient complete the request. Now I have a quick function to start testing with.
```powershell
[void] [Reflection.Assembly]::LoadWithPartialName("System.Diagnostics")
[void] [Reflection.Assembly]::LoadFile("C:\Program Files (x86)\Microsoft System Center 2012 R2 Configuration Manager SDK\Redistributables\Microsoft.ConfigurationManagement.Messaging.dll")
Function check-ContentLocation{
param(
$sitecode="LAB",
$ManagementPoint="SERVER01",
[Parameter(Mandatory=$true)]$PkgID,
[Parameter(Mandatory=$true)]$PkgVer,
[Parameter(Mandatory=$true)]$ADSite
)
# Set up the objects
$httpSender = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Sender.Http.HttpSender
$clr = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ConfigMgrContentLocationRequest
$cmReply = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ContentLocationReply
# Define our SCCM Settings for the message
$clr.SiteCode = $sitecode
$clr.Settings.HostName = $ManagementPoint
$clr.LocationRequest.Package.PackageId = $PkgId
$clr.LocationRequest.Package.Version = $PkgVer
$clr.LocationRequest.ContentLocationInfo.ADSite = $ADSite
# Validate
$clr.Validate([Microsoft.ConfigurationManagement.Messaging.Framework.IMessageSender]$httpSender)
# Send the message
$cmReply = $clr.SendMessage($httpSender)
# Get response
$response = $cmReply.Body.Payload.ToString()
while($response[$response.Length -1] -ne '>'){$response = $response.TrimEnd($response[$response.Length -1])}
[xml]$t = $response
If($t.ContentLocationReply.Sites.Site.LocationRecords.LocationRecord.ServerRemotename){
    $t.ContentLocationReply.Sites.Site.LocationRecords.LocationRecord.ServerRemotename
}Else{Write-Output "!!! ---> No Location found for $PkgId at $ADSite !!!!!"}
}
```
Nothing fancy but if the request has distribution points in the return I get the list and if there are none I get a message with the package id and the AD Site that need to be checked. While this is great for one off checks and how do I check lots of packages? Simple, feed a list of packages to the Config Manager Powershell Cmdlets to get the relevant info and you are ready to go.
```powershell
[void] [Reflection.Assembly]::LoadWithPartialName("System.Diagnostics")
[void] [Reflection.Assembly]::LoadFile("C:\Program Files (x86)\Microsoft System Center 2012 R2 Configuration Manager SDK\Redistributables\Microsoft.ConfigurationManagement.Messaging.dll")
Function check-ContentLocation{
param(
$sitecode="LAB",
$ManagementPoint="SERVER01",
[Parameter(Mandatory=$true)]$PkgID,
[Parameter(Mandatory=$true)]$PkgVer,
[Parameter(Mandatory=$true)]$ADSite
)
# Set up the objects
$httpSender = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Sender.Http.HttpSender
$clr = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ConfigMgrContentLocationRequest
$cmReply = New-Object -TypeName Microsoft.ConfigurationManagement.Messaging.Messages.ContentLocationReply
# Define our SCCM Settings for the message
$clr.SiteCode = $sitecode
$clr.Settings.HostName = $ManagementPoint
$clr.LocationRequest.Package.PackageId = $PkgId
$clr.LocationRequest.Package.Version = $PkgVer
$clr.LocationRequest.ContentLocationInfo.ADSite = $ADSite
# Validate
$clr.Validate([Microsoft.ConfigurationManagement.Messaging.Framework.IMessageSender]$httpSender)
# Send the message
$cmReply = $clr.SendMessage($httpSender)
# Get response
$response = $cmReply.Body.Payload.ToString()
while($response[$response.Length -1] -ne '>'){$response = $response.TrimEnd($response[$response.Length -1])}
[xml]$t = $response
If($t.ContentLocationReply.Sites.Site.LocationRecords.LocationRecord.ServerRemotename){
    $t.ContentLocationReply.Sites.Site.LocationRecords.LocationRecord.ServerRemotename
}Else{Write-Output "!!! ---> No Location found for $PkgId at $ADSite !!!!!"}
}

$pkgs = "LAB00006","LAB00007","LAB00008"
Foreach($pkg in $pkgs){
    $pkginfo = Get-CMPackage -Id $pkg 'select Name, SourceVersion
    if(!($pkginfo)){$pkginfo = Get-CMDriverPackage -Id $pkg 'select Name, SourceVersion}
    if(!($pkginfo)){$pkginfo = Get-CMBootImage -Id $pkg 'select Name, SourceVersion}
    if(!($pkginfo)){$pkginfo = Get-CMOperatingSystemImage -Id $pkg 'select Name, SourceVersion}
    if(!($pkginfo)){$pkginfo = Get-CMSoftwareUpdateDeploymentPackage -Id $pkg 'select Name, SourceVersion}
    check-ContentLocation -PkgID $pkg -PkgVer $pkginfo.SourceVersion -ADSite "YourADSite"
}
```
