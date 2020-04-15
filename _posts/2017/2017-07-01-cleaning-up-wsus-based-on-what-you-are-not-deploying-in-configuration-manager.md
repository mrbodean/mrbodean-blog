---
layout: post
title: Cleaning Up WSUS based on what you are not deploying in Configuration Manager
date: 2017-07-01
tags: ["ConfigMgr","Powershell","SCCM","WSUS"]
---

Let me start with this statement, I wish I had something other than WSUS stuff to talk about. It has been another long week and more issues related to patching. Even with all the other tips I have shared, we experienced major issues getting patches applied. In case you are not aware the [windows update agent can have a memory allocation error .](https://blogs.technet.microsoft.com/configurationmgr/2015/04/15/support-tip-configmgr-2012-update-scan-fails-and-causes-incorrect-compliance-status/) The good news is that is you keep your systems patched there is a hotfix to address the issue on most systems. The bad news is that the patch for the issue was not made available for the Standard Editions of Windows Server 2008 and 2012. If you have these operating systems installed and they are 64 versions; with plenty of memory, you may not see the issue or it may just be transitory and clear up on the next update scan. I am not that lucky and have lots of Windows 2012 Standard servers with 2GB of memory.  The strange part of this is that it seemed like some systems would complete a scan and report a success only to report corruption of the windows update data store. This would cause the next update scan to be a full scan and it would rebuild the local data store and the cycle of issues would start again. The fun part is that when this is occurring if you deploy patches via Configuration Manager the client will fail to identify any patches to apply and report that is compliant for the updates in the deployment. The next successful software update scan would then find the patches missing and the system will return to a non compliance state. (This is justification for external verification of patch installs from what ever product you use to install patches. But that is a story for another day. ) So back to the post from Microsoft on the issue, basically if you can not apply the hot fix you have 2 options.

1.  Move wuauserv (Windows Update Agent) to its own process. (But on systems with less than 4GB of memory this will not gain you much and can be counter productive and impact applications running on the server. )
2.  Cleanup WSUS
For my issue adding memory to the clients was recommended and the Server team to make the change. But one of the joys of working in a large enterprise is that this will take awhile, (not weeks .. months at least). But in the interim, I need to do everything possible to decline updates in WSUS to reduce the catalog size.  At the start of these steps I had ~6200 un-declined updates in WSUS. The guidance I got from Microsoft was to target between 4000 -5000 updates in the catalog. But the lower the number the better off we would be.

Step one review the products and categories that we sync. This was easy because we already review this routinely. There was not much to change but I did trim a few and could decline a 100 or so updates. Not much everything helps.

Step two review the superseded updates. Due to earlier  patching issues our patching team had requested that we keep superseded updates for 60 days. Now this was before the updates had moved to the cumulative model and at this point ensuring the current security patches were applying was critical. (Thank you wanttocry and notpetya) So I checked to see which updates had been superseded for 30 days. I found ~1300, checking for less then 30 days only found 1 more. Big win there so after declining those the WSUS catalog was down to ~4700 updates. That got us under the upper limit of the suggested target. After triggering scans on the systems having issues and reviewing the status, it did help but not enough to call it significant improvement.

Step three break out the coffee and dig in. Wouldn't be great to see what patches had not been declined that and  are not deployed in Configuration Manager. Easy enough to see what is not deployed in the console for SCCM but you have to look up the update in WSUS to see if it has been declined. At this point I am on the hook to stay up and monitor the patching installs and help the patching team; there are a couple of hours to kill between the end of the office day and when the bulk of our patch installs occur. So I started poking around to see what I could do to automate the comparison between Configuration Manager and WSUS. Our good friend PowerShell to the rescue. First thing is to get the patches from SCCM.  This
```powershell
$SCCMServer = "YourServerHere"
$SiteCode = "YourSiteCodeHere"
Get-WmiObject -Namespace "root\SMS\site_$SiteCode" -class SMS_SoftwareUpdate -ComputerName $SCCMServer'select -First1
```
This connects to your server and gets all the patches listed in the console and selects the first one so you can take a look at all the properties. I am excluding a few with identifying information but you will see something similar.
```
ArticleID                          : 949189
BulletinID                         :
CategoryInstance_UniqueIDs         : {UpdateClassification:cd5ffd1e-e932-4e3a-bf74-18bf0b1bbd83, Product:ba0ae9cc-5f01-
                                     40b4-ac3f-50192b5d6aaf, Locale:0}
CI_ID                              : 16783644
CI_UniqueID                        : 2e6d75eb-f486-4e40-909d-615e43de537f
CIType_ID                          : 8
CIVersion                          : 101
CreatedBy                          :
CustomSeverity                     : 0
CustomSeverityName                 :
DateCreated                        : 20140104043952.000000+000
DateLastModified                   : 20170629045030.980000+000
DatePosted                         : 20080408170000.000000+000
DateRevised                        : 20080408170000.000000+000
EffectiveDate                      :
EULAAccepted                       : 2
EULAExists                         : False
EULASignoffDate                    :
EULASignoffUser                    :
ExecutionContext                   : 0
IsBundle                           : True
IsContentProvisioned               : False
IsDeployable                       : False
IsDeployed                         : False
IsDigest                           : True
IsEnabled                          : True
IsExpired                          : True
IsHidden                           : False
IsLatest                           : True
IsMetadataOnlyUpdate               : False
IsOfflineServiceable               : True
IsQuarantined                      : False
IsSuperseded                       : False
IsUserDefined                      : False
IsVersionCompatible                :
LastModifiedBy                     :
LastStatusTime                     : 20170630172253.503000+000
LocalizedCategoryInstanceNames     : {Updates, Windows Server 2008, }
LocalizedDescription               : Install this update to resolve an issue where a replica domain controller may sile
                                     ntly fail to receive updates to some object attributes during the inbound replicat
                                     ion, when the replica domain controller is running Windows Server 2008 with the Ja
                                     panese Language locale installed. After you install this item, you may have to res
                                     tart your computer.
LocalizedDisplayName               : Update for Windows Server 2008 x64 Edition (KB949189)
LocalizedEulas                     :
LocalizedInformation               :
LocalizedInformativeURL            : http://go.microsoft.com/fwlink/?LinkId=111973
LocalizedPropertyLocaleID          : 9
MaxExecutionTime                   : 600
ModelID                            : 16783644
ModelName                          : Site_56FE87C5-F355-45F9-BE43-BE8D575809F4/SUM_2e6d75eb-f486-4e40-909d-615e43de537f
NumMissing                         : 0
NumNotApplicable                   : 59326
NumPresent                         : 0
NumTotal                           : 59568
NumUnknown                         : 242
PercentCompliant                   : 100
PermittedUses                      : 0
PlatformCategoryInstance_UniqueIDs : {}
PlatformType                       : 1
RequiresExclusiveHandling          : False
RevisionNumber                     : 101
SDMPackageLocalizedData            :
SDMPackageVersion                  : 101
SDMPackageXML                      :
SecuredScopeNames                  : {}
SedoObjectVersion                  :
Severity                           : 0
SeverityName                       :
Size                               :
```
Looks great and lots of thing to use to select patches to check on. However if you use query or filter you will find that a lot of those properties are [lazy properties]({% post_url /2015/2015-10-07-Lazy-is-as-Lazy-does %}). If you pull all the properties for the 1000s of patches the script will run a looooong time. However if you so a select on the object you will get the value reported from the query and you can select what you want using a where-object in PowerShell.  I decided that the following properties would allow me to evaluate the patches: LocalizedDisplayName, CI_UniqueID, IsDeployed, NumMissing
```powershell
$SCCMServer = "YourServerHere" 
$SiteCode = "YourSiteCodeHere" 
$cmpatches = Get-WmiObject -Namespace "root\SMS\site_$SiteCode" -class SMS_SoftwareUpdate -ComputerName $SCCMServer'LocalizedDisplayName, CI_UniqueID, IsDeployed, NumMissing
```
Now to get patches that are not deployed and are not required
```powershell
$cmpatches'?{($_.IsDeployed -eq $false) -and ($_.NumMissing -eq 0)}'out-gridview
```
And patches that are not deployed and are required
```powershell
$cmpatches'?{($_.IsDeployed -eq $false) -and ($_.NumMissing -ne 0)}'out-gridview
```
Using this information you are determine a criteria to select the patches to decline. I  settled on patches that are not required and not deployed and have been available for more then 30 days. You can download the script from [https://gallery.technet.microsoft.com/Decline-Update-in-WSUS-e934565f](https://gallery.technet.microsoft.com/Decline-Update-in-WSUS-e934565f)
```powershell
Function Unpublish-UnUsedCMPatches{
    [CmdletBinding()]
    Param(
        # WSUS Server Name
        [Parameter(Mandatory=$true,
                   ValueFromPipelineByPropertyName=$true)]
        [string]$WsusServer,
        # Use SSL for the WSUS connection. Default value is $False
        [Parameter()]
        [bool]$UseSSL = $False,
        # Port to use for WSUS connection. Default value is 8530
        [Parameter()]
        [int]$PortNumber = 8530,
        # Configuration Manager Site Server with SMS Provider role
        [Parameter(Mandatory=$true,
                   ValueFromPipelineByPropertyName=$true)]
        [string]$SCCMServer,
        # Configuration Manager Site Code
        [Parameter(Mandatory=$true,
                   ValueFromPipelineByPropertyName=$true)]
        [string]$SiteCode,
        # Decline Updates. Defaults to False for safety
        [Parameter()]
        [switch]$Decline
    )
    $outPath = Split-Path $script:MyInvocation.MyCommand.Path
    $outDeclineList = Join-Path $outPath "DeclinedUpdates.csv"
    $outDeclineListBackup = Join-Path $outPath "DeclinedUpdatesBackup.csv"
    if(Test-Path -Path $outDeclineList){Copy-Item -Path $outDeclineList -Destination $outDeclineListBackup -Force}
    "UpdateID, RevisionNumber, Title, KBArticle, SecurityBulletin, LastLevel" ' Out-File $outDeclineList
    $cmpatchlist = Get-WmiObject -Namespace "root\SMS\site_$SiteCode" -class SMS_SoftwareUpdate -ComputerName $SCCMServer'Select-Object LocalizedDisplayName, CI_UniqueID, IsDeployed, NumMissing
    $cmpatchlistcount = $cmpatchlist.Count
    If($cmpatchlistcount -eq 0){ 
        Throw "Error Collecting patches from $SCCMServer"
        return
    }
    "Found $cmpatchlistcount updates on $SCCMServer"
    $cmpatchlist = $cmpatchlist'?{($_.IsDeployed -eq $false) -and ($_.NumMissing -eq 0)}
    $cmpatchlistcount = $cmpatchlist.Count
    "Found $cmpatchlistcount updates on $SCCMServer that are not deployed and not required. These will be evaluated to determine if they are older then 30 days and not declined. "
    #Connect to the WSUS 3.0 interface.
    [reflection.assembly]::LoadWithPartialName("Microsoft.UpdateServices.Administration") ' out-null
    If($? -eq $False)
    {       Write-Warning "Something went wrong connecting to the WSUS interface on $WsusServer server: $($Error[0].Exception.Message)"
            $ErrorActionPreference = $script:CurrentErrorActionPreference
            Return
    }
    $WsusServerAdminProxy = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer($WsusServer,$UseSSL,$PortNumber);
    if($i){Remove-Variable i}
    If($updatesDeclined){Remove-Variable updatesDeclined}
    $updatesDeclined =0
    Foreach($item in $cmpatchlist){
        Remove-Variable patch -ErrorAction silentlycontinue
        $i++
        $percentComplete = "{0:N2}" -f (($i/$cmpatchlistcount) * 100)
        Write-Progress -Activity "Decline Unused Updates" -Status "Checking if update #$i/$cmpatchlistcount - $($item.LocalizedDisplayName)" -PercentComplete $percentComplete -CurrentOperation "$($percentComplete)% complete"
        Try{
            $patch = $WsusServerAdminProxy.getUpdate([guid]$item.CI_UniqueID)
            if(($patch.IsDeclined -eq $false) -and ( ($patch.CreationDate) -lt (get-date).AddDays(-30) ) ){
                Write-Progress -Activity "Decline Unused Updates" -Status "Declining update #$i/$cmpatchlistcount - $($item.LocalizedDisplayName)" -PercentComplete $percentComplete -CurrentOperation "$($percentComplete)% complete"
                "$($patch.Id.UpdateId.Guid), $($patch.Id.RevisionNumber), $($patch.Title), $($patch.KnowledgeBaseArticles), $($patch.SecurityBulletins), $($patch.HasSupersededUpdates)" ' Out-File $outDeclineList -Append
                If($Decline){$patch.Decline()}
                $updatesDeclined++
            }else{
                Write-Progress -Activity "Decline Unused Updates" -Status "Update #$i/$cmpatchlistcount - $($item.LocalizedDisplayName) is already declined or was recieved withing the last 30 days" -PercentComplete $percentComplete -CurrentOperation "$($percentComplete)% complete"
            }
        }catch{
        #"$($item.LocalizedDisplayName) was not found in WSUS"
        Write-Progress -Activity "Decline Unused Updates" -Status "Update #$i/$cmpatchlistcount - $($item.LocalizedDisplayName) was not found in WSUS" -PercentComplete $percentComplete -CurrentOperation "$($percentComplete)% complete"
        }
    }
    If($Decline -eq $False){"$updatesDeclined updates would have been declined"}else{"$updatesDeclined updates were declined"}
}
```
Another ~2500ish declined and now the WSUS catalog is down to ~2200  patches. This did help improve the scans and patch deployments for all but the servers with 2GB of memory. But the patches for them can be delivered via a software distribution package until all the memory upgrades are completed.