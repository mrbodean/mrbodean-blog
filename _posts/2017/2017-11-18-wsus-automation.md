---
layout: post
title: WSUS Automation
date: 2017-11-18
tags: ["ConfigMgr","Powershell","SCCM"]
---

So there is a new blog that SCCM admins should take a peek at. Bryan Dam is my hero for having the time to combine most WSUS maintenance into one script.[ http://damgoodadmin.com/2017/11/05/fully-automate-software-update-maintenance-in-cm/](http://damgoodadmin.com/2017/11/05/fully-automate-software-update-maintenance-in-cm/) I encourage you to go take a look at his blog and the presentation he did about the script. [https://www.facebook.com/DeploymentResearch/videos/2141149579244220/](https://www.facebook.com/DeploymentResearch/videos/2141149579244220/)

Seriously good info. As a thank you to Bryan for saving me time I am going to respond to a statement in the blog post on his script. "Once an update has been declined in WSUS and synced to Configuration Manager I honestly don't know how you bring it back.  I'm ... sure ... there's a way somehow."

Well as someone forced into aggressive declines to keep the WSUS catalog to a reasonable size I was forced to learn how. Once you know how to restore a declined update you can decline without fear. So how do you do it? Approve the update in WSUS and sync. But you say it is not that simple, "I tried it and it did not show up". Well when has the WSUS and Configuration Manager interface ever been simple. The trick is that the sync must be a full sync not a delta sync. To trigger a full update run this powershell on your primary site server
```Powershell
$siteserver = "YourServer"
$sitecode = "XXX" #Your 3 Digit Site Code
$SUP = [wmiclass]("\\$siteserver\root\SMS\Site_$sitecode:SMS_SoftwareUpdate")
$Params = $SUP.GetMethodParameters("SyncNow")
$Params.fullSync = $true
$Return = $SUP.SyncNow($Params)
```
One other thing to note when re-approving in WSUS. Unapproved is an approved status for SCCM. Basically everything that is not declined will sync to SCCM. By approving the patch as unapproved you will return it to the normal state that SCCM maintains. If you have any systems that patch via WSUS directly using your Software Update Point then approve as needed as it will not impact SCCM.