---
layout: post
title: Distribution Points not reporting Usage data
date: 2017-02-15
tags: ["ConfigMgr","Powershell","SCCM"]
---

I am scratching my head about the what happened on this issue, but let me explain. We had a issue come up in our SCCM RAP. (If you are a Premier Customer, I highly recommend the RAP as a service. Get it, Use it, and Use it often.) There were about 30ish distribution points that were not reporting usage stats for our 2012 R2 Configuration Manager site. A quick check showed that the distribution points where alive and well but that the scheduled task that reports the usage statistics was gone.  If you need a good primer on how to check out a distribution point see this post from [Scott's IT Blog](https://blogs.technet.microsoft.com/scotts-it-blog/2015/01/19/verifying-a-distribution-point-has-installed-successfully/). To resolve this I simply exported the task from a working distribution point and imported on the systems where it was missing. To be honest though it did not get fixed everywhere. A couple of months go by and we rerun the RAP and look at the results. Now there are over 300 distribution points not reporting statistics because of a missing scheduled task. This makes me beg and plead to speed up our upgrade project for Current Branch. But until then I have to keep everything going, so a little powershell to the rescue.

First I choose to export the existing scheduled task from a working server and save it as c:\temp\Content Usage.xml

The Rap web site is great for reporting the issue but not so much for getting the details in a way that is easily usable in a script. So here is a SQL query to identify distribution points not reporting usage date.
```SQL
Select  DP.ServerName
From v_DistributionPoints AS DP
Where Not Exists 
    (Select DPStats.PkgServer from v_DPUsageSummary As DPStats
     Where DPStats.PkgServer = DP.ServerName)
```
This will give you a list of server names that you can save in a file. Now for the powershell to recreate the scheduled task and run it.
```powershell
$servers = get-content c:\temp\Servers.txt
foreach($server in $servers){
  $server
  schtasks /create /xml "c:\temp\Content Usage.xml" /TN "Microsoft\Configuration Manager\Content Usage" /s $server
  schtasks /run /TN "Microsoft\Configuration Manager\Content Usage" /s $server
}
```
Give it a little time and rerun the SQL query to verify that the systems are reporting usage data and are being removed from the report.