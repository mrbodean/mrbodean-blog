---
layout: post
title: Reporting on the Total Physical Memory installed vs the Memory visible to the OS
date: 2017-01-19
tags: ["ConfigMgr","SCCM","SQL"]
---

Recently I was asked if SCCM could report on the total physical memory installed. No problem there is even a built in report for that, I replied. No the requester explained, on systems with a 32 OS installed those reports only show the max memory the OS can see. So we sit down and go through the Resource Explorer and find that we are collecting the Win32_PhysicalMemory Class from WMI as part of our hardware inventory. A quick little query gets the info for that can be made into a report.
```SQL
Select RSystem.Name0 As Name, RSystem.Netbios_Name0 As 'NetBios Name', 
SUM(PhyMem.Capacity0) As 'Total Physical Memory Installed (MB)',
OS.TotalVisibleMemorySize0 As 'OS Total Memory(MB)', 
GSystem.SystemType0 As 'System Type',
OS.Caption0 As OS, Processor.Name0 As Processor,PC.Model0 As Model
From v_R_System As RSystem
Inner Join v_GS_PHYSICAL_MEMORY As PhyMem on PhyMem.ResourceID = RSystem.ResourceID
Inner Join v_GS_OPERATING_SYSTEM As OS on OS.ResourceID = RSystem.ResourceID
Inner Join v_GS_SYSTEM As GSystem on GSystem.ResourceID = RSystem.ResourceID
Inner Join v_GS_PROCESSOR As Processor On Processor.ResourceID = RSystem.ResourceID
Inner Join v_GS_COMPUTER_SYSTEM As PC On PC.ResourceID = RSystem.ResourceID
Group By RSystem.Name0, RSystem.Netbios_Name0, OS.TotalVisibleMemorySize0,
GSystem.SystemType0,OS.Caption0, Processor.Name0,PC.Model0
Order By RSystem.Name0
```
