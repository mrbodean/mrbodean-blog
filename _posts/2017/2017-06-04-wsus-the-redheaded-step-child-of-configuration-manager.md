---
layout: post
title: WSUS the Redheaded step child of Configuration Manager
date: 2017-06-04
tags: ["ConfigMgr","SCCM"]
---

So like a lot of people I drank the kool aid for WSUS and Config Manager. Install the feature let SCCM configure it and forget the WSUS console exists. As long as you do some occasional maintenance it just works. Then the cumulative patches came along and every month this year has had 5-6 days each devoted to "fixing" the WSUS\SUP servers. I know I am not alone in fighting high CPU spikes while patching. I have added more memory and CPU to the servers, it helped but the next month the issue returned. Open a case with Microsoft and got a very intensive lesson on how to do maintenance the right way. Which if you need to learn check out [The complete guide to Microsoft WSUS and Configuration Manager SUP maintenance ](https://blogs.technet.microsoft.com/configurationmgr/2016/01/26/the-complete-guide-to-microsoft-wsus-and-configuration-manager-sup-maintenance/)and then follow it. But even after all that the issue started to reoccur as my patching team was dealing with the stragglers from the last patching round.

So I opened another case up with the wonderful folks at premier support and we start looking. This time around I would just getting spikes in CPU that would clear up after a hour or 6. As we checked and rechecked everything we were seeing that as few as 50ish connections to the WSUS site would spike the CPU utilization up to 80-90%. Prior to all these issue I would see an average CPU utilization on these servers of 30-40%. While there would be spikes during heavy patching periods they were also accompanied by large numbers of connections to the WSUS site. Using this as justification to finally clean out some obsolete products from the catalog,(Yes Server 2003 was still in there), I unchecked a few products and synced. After running the cleanup, reindex, and decline process; Still no improvement. After looking at the calendar and seeing the next Patch Tuesday coming quickly, I though well if it is going to be another crappy patch cycle lets try just doing security patches and kick everyone one else out of the pool. Well the Updates classification has the largest number of updates in my environment. (This may not be the case in yours.)  So I unchecked the classification and synced. Wow, performance dropped back to normal. To be sure, I triggered a couple of thousand update scans. I was able to get several hundred active connections and the CPU never spiked over 60% and was averaging ~30% utilization. To double check that this was truly the cause, I added the Updates classification back and synced. The sync took about 2 hours to finish and the CPU utilization started spiking again. This time 90-100 %, quick dig in and look at the root cause.

So I start searching through the updates in WSUS and comparing to what is being deployed via Config Manager. WSUS still has lots and Server 2003 updates and I just removed them, why are they still approved? I even found some XP and 2000 Updates approved in WSUS and they have been long gone. But the updates were approved and the WSUS server was diligently querying them to see if they applied and updating the status for them as well. So based on the assumption all those old products were increasing catalog to the point that performance was suffering, I started looking for a way to clean up.  *While I am going to talk about my script and hope you use it, Full credit to the Decline-SupersededUpdates.ps1 linked in [The complete guide to Microsoft WSUS and Configuration Manager SUP maintenance ](https://blogs.technet.microsoft.com/configurationmgr/2016/01/26/the-complete-guide-to-microsoft-wsus-and-configuration-manager-sup-maintenance/)and to [Automatically Declining Itanium Updates in WSUS ](https://gallery.technet.microsoft.com/scriptcenter/Automatically-Declining-a4fec7be)as the basis for how to do all this cleanup via powershell.

Now back to the investigation, I still wanted to figure out why all these updates had been approved. After lots of checking and comparing between various sites I found that my top level WSUS server for SCCM had the default auto approval rule enabled in WSUS. Well that explains the why but now for the clean up. To help Identify the updates I wanted to decline I used this powershell
<pre class="lang:ps decode:true">[string]$WsusServer = ([system.net.dns]::GetHostByName('localhost')).hostname
[bool]$UseSSL = $False
[int]$PortNumber = 8530
[reflection.assembly]::LoadWithPartialName("Microsoft.UpdateServices.Administration") ' out-null
$WsusServerAdminProxy = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer($WsusServer,$UseSSL,$PortNumber)
$AllUpdates =  $WsusServerAdminProxy.GetUpdates() ' ?{-not $_.IsDeclined}
$AllUpdates'Out-GridView</pre>
This will grab all updates that are not declined and send them to a Gridview window. I like this because when wsus is overworked the console can timeout frequently and I find it easier to search through all the updates this way. A few things to remember about this, This code assumes you are running on the WSUS server you are checking. It can be run remotely on any system that has the management tool install. You will need to adjust the variables to match your environment.

If you get timeouts with this then your wsus server needs some love. You can retry but if you get timeouts 2 or 3 times stop and go read [the complete guide to Microsoft WSUS and Configuration Manager SUP maintenance](https://support.microsoft.com/en-us/help/4490644/complete-guide-to-microsoft-wsus-and-configuration-manager-sup-maint). Follow those steps and come back and try again.

Once you get the GridView window start searching for updates that can be declined. For example search for XP and see what you get. Here is what I found on one of my servers
![wsus_xpsearch]({{site.url}}/{{site.baseurl}}/media/wsus_xpsearch-1024x526.jpg)
Lots and lots of XP updates. What I found is that even when you stop syncing the product the updates already in the catalog stay until you decline them. Why does the matter you ask? While the clients do not get them sent to them the wsus server has to process the updates in queries when a client request a scan. In my case a server with plenty of CPU and Memory using a full SQL install could only handle ~50 scan requests before getting overworked. After declining all the old unwanted update performance returned to normal.

Using the variable $allupdates from the powershell above I created several rules to identify and decline updates. Now this is what could be declined in my environment. YOU MUST EVALUATE WHAT CAN BE DECLINED IN YOUR ENVIRONMENT. I am posting these a examples of what I did and how I cleaned up my environment. If you copy what I did and find that you need the updates, all is not lost, just approve the update again and it will be available again.
```powershell
#Delcine Itanium Updates
$ItaniumUpdates = $AllUpdates ' ?{$_.Title -match 'ia64'itanium'}
If($ItaniumUpdates)
{
    "Found $($ItaniumUpdates.Count) Itanium Updates that will be declined"
    $ItaniumUpdates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Itanium Updates found that needed declining. "}
#Delcine Windows 2000 Updates
$W2kUpdates = $AllUpdates ' ?{$_.ProductTitles -match 'Windows 2000'}
If($W2kUpdates)
{
    "Found $($W2kUpdates.Count) Windows 2000 Updates that will be declined"
    $W2kUpdates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Windows 2000 Updates found that needed declining. "}
#Delcine Windows XP Updates
$XPUpdates = $AllUpdates ' ?{$_.ProductTitles -match 'Windows XP'}
If($XPUpdates)
{
    "Found $($XPUpdates.Count) Windows XP Updates that will be declined"
    $XPUpdates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Windows XP Updates found that needed declining. "}
#Delcine Windows Server 2003 Updates
$2K3Updates = $AllUpdates ' ?{$_.ProductTitles -match 'Windows Server 2003'}
If($2K3Updates)
{
    "Found $($2K3Updates.Count) Windows Server 2003 Updates that will be declined"
    $2K3Updates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Windows Server 2003 Updates found that needed declining. "}
#Delcine Language Pack Updates
$2K3Updates = $AllUpdates ' ?{$_.Title -match 'Language Pack'Language Interface Pack'Language Features'}
If($2K3Updates)
{
    "Found $($2K3Updates.Count) Language Pack Updates that will be declined"
    $2K3Updates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Windows Language Pack Updates found that needed declining. "}
#Delcine Vista Only Updates
$VistaUpdates = $AllUpdates ' ?{$_.ProductTitles -eq 'Windows Vista' -and $_.ProductTitles.count -eq 1}
If($VistaUpdates)
{
    "Found $($VistaUpdates.Count) Windows Vista Updates that will be declined"
    $VistaUpdates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Windows Vista Updates found that needed declining. "}
#Delcine Win10 x86 Updates
$Win10x86Updates = $AllUpdates ' ?{$_.ProductTitles -eq 'Windows 10' -and $_.Title -match 'for x86'}
If($Win10x86Updates)
{
    "Found $($Win10x86Updates.Count) Win10 x86 Updates that will be declined"
    $Win10x86Updates ' %{
        Write-Progress -Activity "Declining Updates" -Status "Declining $($_.KnowledgebaseArticles) - $($_.Title)"
        $_.Decline()
    }
}Else
{"No Win10 x86 Updates found that needed declining. "}
```
Now with all that being said I wish that I could give you a definitive recommendation on what number of un-declined update with cause you issues but I don't because every environment different. What I can say is now we must monitor the wsus catalog and ensure the our maintenance processes now ensure that unused and unwanted updates are declined.

Here is the full cleanup script I used to get back to normal

{% gist 0fd0a527c3f7ad359c79e9e2cb3c7a1c %}
