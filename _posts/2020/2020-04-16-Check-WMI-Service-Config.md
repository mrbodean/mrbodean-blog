---
layout: post
title: Check WMI Service Configuration
date: 2020-04-16
tags: ["ConfigMgr","Powershell"]
---

Just a quick share on how to work with command line output in powershell. 
I was working with an organization that is still running Windows 7 and they were implementing some applications that caused some concern about WMI performance. This [post from the Microsoft Ask Performance Team](https://techcommunity.microsoft.com/t5/ask-the-performance-team/wmi-how-to-troubleshoot-high-cpu-usage-by-wmi-components/ba-p/375494) details how to move the service into its own process. 
I wrote this powershell script to check the configuration of the service as in Configuration Manager Compliance Item.

{% gist cc55d33c37a9ce0cf2e8851c2fc30223 %}

If the return is 20 then the service is configured as a shared process. 
If the return is 10 then the service is configured as its own process. 
If the return is 30 then the OS does not have the issue. 

I set my CI to be compliant if the value returned was 10 or 30. Armed with the compliance data we were able to target systems for remediation. 