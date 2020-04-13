---
layout: post
title: The Evolution of a Script or why I am constantly refactoring my code.
date: 2015-10-12
tags: ["Powershell","SCCM"]
---

I have a confession to make, I hate reinventing the wheel when it comes to scripting. Most of the time, there are functioning examples of what I need to do readily available. All I need to do is just ask Bing or Google. Sometimes I piece together functionality from numerous scripts to make a new script that fills the need for the task at hand. This is the story of one such script.

Below you will find the gist of the Check-TSDependencies script in all its ugly glory.  As you can plainly see no though of reusability. Definitely not a tool script. If you stretch your imagination perhaps it could be called a controller script. But to be this was a get the job done; save MrBoDean some time; throw away script.

{% gist c66994593458a196001c %}


Well as I mentioned in [a previous post]({% post_url /2015/2015-10-05-Get-the-Source-location-for-all-packages-in-a-SCCM-Task-Sequence %}/)  I have been working on improving the processes we use at work to manage SCCM and this ugly little script ended up filling a real need. SCCM has great reporting in the console, if you want to see the status of all distribution points for a single package or the status of all packages on a single distribution point. But trying to determine the status of 30-40 packages quickly does not happen very quickly.  That is until I started running this script.

Then the need for improvements came a long. "This is great for one distribution point but I keep needing to 2 or 3 or 10 ..."; "This is great for troubleshooting after we have a issue, how can it be used proactively?"; "Displaying the status on the screen is ok but can you send a email of the status? Oh and I only want the email if there are failures." and so on ...

It quickly became apparent that this needed to evolve. No problem this script is so useful all it needs is parameters for prompts and hard coded packages and SCCM config. Maybe pretty it up and have it use some [best practices](https://github.com/PoshCode/PowerShellPracticeAndStyle). But when I take some time and really analyze the code that would not be very effective. The core functionality of the script is the wmi query and translating the results into easy to understand messages. The interactive prompts for the distribution point and the task sequence name are defiantly candidates for parameters. But the interactive portion would have to go or be placed in a controller script. But a little work with the SCCM comandlets and the PowerShell pipeline and the controller may not be needed.

The end result
```
<#
.Synopsis
   Get the status of packages used in a Config Manager Task Sequence on a specified Distribution Point(s)
.EXAMPLE
    Get-TSContentDeploymentStatus -SiteSystemServerName DEVSCCMDP1.my.domain.com -SiteCode DEV -TaskSequencePackageId DEV001A2
.EXAMPLE
    "DEVSCCMDP1.my.domain.com","DEVSCCMDP2.my.domain.com"'Get-TSContentDeploymentStatus -SiteCode DEV -TaskSequencePackageId DEV001A2
.NOTES
    Author
        Jon Warnken
        @MrBoDean
        jon.warnken@gmail.com
#>
function Get-TSContentDeploymentStatus
{
    [CmdletBinding()]
    #Requires -Version 3.0
    #Requires -Modules ConfigurationManager
    Param
    (
        # The Site System Server Name (AKA Distribution Point Computer Name).
        # This must be the FQDN
        [Parameter(Mandatory=$true,
                   ValueFromPipeline=$true,
                   ValueFromPipelineByPropertyName=$true)]
        [Alias("DistributionPoint","DP")]
        [String[]]$SiteSystemServerName,
        # The Site Code for the Config Manager Site you wish to perform the check against.
        [Parameter(Mandatory=$true)]
        [String]$SiteCode,
        #Package Id for the Task Sequence
        [Parameter(Mandatory=$true)]
        [Alias("PackageId","ID")]
        [String]$TaskSequencePackageId
    )

    Begin
    {
        If((Get-Location).Drive.Name -ne $SiteCode){
            Try{Set-Location -path "$($SiteCode):" -ErrorAction Stop}
            Catch{Throw "Unable to connect to Site $SiteCode. Ensure that the Site Code is correct and that you have access."}
        }#If
        $CMServerName = $(Get-CMSite -SiteCode $SiteCode'select ServerName).ServerName
        $CMSite = $SiteCode
        [xml]$TS = Get-CMTaskSequence -TaskSequencePackageId $TaskSequencePackageId' select -ExpandProperty Sequence
        $rpkg = $ts.sequence.referenceList.reference.package'Select-Object -Unique
    }#Begin
Process{
      Foreach($pkg in $rpkg){
            $pkgname = (Get-CMPackage -Id $pkg 'Select-Object Name).Name
            if(!($pkgname)){$pkgname = (Get-CMDriverPackage -Id $pkg 'Select-Object Name).Name}
            if(!($pkgname)){$pkgname = (Get-CMBootImage -Id $pkg 'Select-Object Name).Name}
            if(!($pkgname)){$pkgname = (Get-CMOperatingSystemImage -Id $pkg 'Select-Object Name).Name}
            if(!($pkgname)){$pkgname = (Get-CMSoftwareUpdateDeploymentPackage -Id $pkg 'Select-Object Name).Name}
            #Write-Host "Get-WmiObject -NameSpace Root\SMS\Site_$SiteCode -Class SMS_DistributionDPStatus -ComputerName $CMServerName -Filter PackageID='$pkg' And Name='$SiteSystemServerName'"
            $query = Get-WmiObject -NameSpace Root\SMS\Site_$SiteCode -Class SMS_DistributionDPStatus -ComputerName $CMServerName -Filter "PackageID='$pkg' And Name='$SiteSystemServerName'" ' Select Name, MessageID, MessageState, LastUpdateDate
            If ($query -eq $null){
                $Status = "Failed"
                $Message = "No status found! Ensure the package content has been deployed to this distribution point"
                $DPName = $SiteSystemServerName.Trim('{}')
            }else{
                Foreach ($objItem in $query){
                    $DPName = $null
                    $DPName = $objItem.Name
                    $UpdDate = [System.Management.ManagementDateTimeconverter]::ToDateTime($objItem.LastUpdateDate)
                    switch ($objItem.MessageState){
                        1{$Status = "Success"}
                        2{$Status = "In Progress"}
                        3{$Status = "Unknown"}
                        4{$Status = "Failed"}
                    }#Switch
                    switch ($objItem.MessageID){
                        2303{$Message = "Content was successfully refreshed"}
                        2323{$Message = "Failed to initialize NAL"}
                        2324{$Message = "Failed to access or create the content share"}
                        2330{$Message = "Content was distributed to distribution point"}
                        2354{$Message = "Failed to validate content status file"}
                        2357{$Message = "Content transfer manager was instructed to send content to Distribution Point"}
                        2360{$Message = "Status message 2360 unknown"}
                        2370{$Message = "Failed to install distribution point"}
                        2371{$Message = "Waiting for prestaged content"}
                        2372{$Message = "Waiting for content"}
                        2380{$Message = "Content evaluation has started"}
                        2381{$Message = "An evaluation task is running. Content was added to Queue"}
                        2382{$Message = "Content hash is invalid"}
                        2383{$Message = "Failed to validate content hash"}
                        2384{$Message = "Content hash has been successfully verified"}
                        2391{$Message = "Failed to connect to remote distribution point"}
                        2398{$Message = "Content Status not found"}
                        8203{$Message = "Failed to update package"}
                        8204{$Message = "Content is being distributed to the distribution Point"}
                        8211{$Message = "Failed to update package"}
                    }#Switch
                }#Foreach
            }#if else 
            [PSCustomObject]@{
                'Name' = $pkgname
                'PackageID' = $Pkg
                'Distribution Point'= $DPName
                'Status' = $Status
                'Message' = $Message
            }#[PSCustomObject]@
      }#foreach
   }#process
End{}
}
```
Also available [https://gallery.technet.microsoft.com/Get-the-content-deployment-a2aab406](https://gallery.technet.microsoft.com/Get-the-content-deployment-a2aab406) and [https://github.com/mrbodean/Technet/blob/master/Powershell/Get-TSContentDeploymentStatus/Get-TSContentDeploymentStatus.ps1](https://github.com/mrbodean/Technet/blob/master/Powershell/Get-TSContentDeploymentStatus/Get-TSContentDeploymentStatus.ps1)

&nbsp;