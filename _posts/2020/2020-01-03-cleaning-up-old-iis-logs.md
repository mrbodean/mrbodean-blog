---
layout: post
title: Cleaning up old IIS logs
date: 2020-01-03
tags: ["ConfigMgr","Powershell"]
---

Eventually every Configuration Manager Admin will need to cleanup IIS logs. It might take years or months to get there but left unchecked the IIS logs just keep growing. For those unaware, IIS will create a log file for each day. Depending on how active your site is these logs are several KB to hundreds of MB in size, and there is no builtin mechanism to archive or clean them up. After helping automate this again, I though it was time to share my variation on how to clean up the file.  I use a simple powershell script to remove any log files over 30 days old. This is generally sufficient for troubleshooting but feel free to adjust the time period to meet your needs.


{% gist 2a407c2f23c7ba6304508feca158af03 %}

To update the time period change -30 to the number of days that you want to preserve, on the line

`$oldest = (get-date).AddDays(-30)`

If you run this as a scheduled task the cleanup will happen automatically. 