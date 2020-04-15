---
layout: post
title: Quickly get system stats for Software Update Point\Management Point
date: 2017-05-15
tags: ["ConfigMgr","SCCM"]
---

Like most of us out here is SCCM land Patch Tuesday generally means that you will see a performance for the WSUS app pool that you will need to ensure does not lead to issues. Here is a quick PowerShell command  to get the CPU utilization and the current connections the websites. I use it for monitoring both management points and software update points. Because it is using performance counters it is easy to add other counters that are relevant to your own environment.
```powershell
get-counter -Counter "\Processor(_total)\% Processor Time","\Web Service(*)\Current Anonymous Users","\Memory\% Committed Bytes in Use","\Memory\Available MBytes"
```
get-counter supports remote systems with the -ComputerName parameter so to check Multiple systems use
```powershell
"Server01","Server02","Server03"|%{get-counter -Counter "\Processor(_total)\% Processor Time","\Web Service(*)\Current Anonymous Users","\Memory\% Committed Bytes in Use","\Memory\Available MBytes" -ComputerName $_}
```