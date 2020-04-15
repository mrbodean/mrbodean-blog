---
layout: post
title: Software Update Troubleshooting - Finding the Problem Children
date: 2017-06-25
tags: ["ConfigMgr","SCCM","SQL"]
---

It can seem like a never ending struggle to keep Configuration Manager clients healthy and ready to install software and patches. After fighting with WSUS the past few patch cycles, I have been sending time drilling into the client side issues. [Eswar Koneti has a post](http://eskonr.com/2016/09/sccm-configmgr-software-update-scan-failed-onsearchcomplete-failed-to-end-search-job-error-0x80072ee2/) that has a great sql query to help identify clients that are not successfully completing a software update scan.  Eswar's query reports the last error code as it is stored in SQL as a  decimal, I find it helpful to convert it to hex as that is what you will see in the client log files. (This makes your googlefu more efficient.) Using Eswar's query as a base, I created this query to help focus of the problem areas.
```SQL
select convert(VARBINARY(8),uss.lasterrorcode)as HexErrorCode,uss.LastErrorCode, count(uss.LastErrorCode) as 'Count'
from v_r_system sys
 inner join v_updatescanstatus uss on uss.ResourceId=sys.ResourceID
 where uss.lasterrorcode!='0'
Group By uss.LastErrorCode
order by 'Count'
```
![SQL_CountofScanErrors_Grouped]({{site.url}}/media/software-update-troubleshooting-finding-the-problem-children/SQL_CountofScanErrors_Grouped.jpg)

This gives you report of the number of systems that are experiencing the same error. A small modification allows you focus in on specific client populations. For example to just report on servers
```SQL
select convert(VARBINARY(8),uss.lasterrorcode)as HexErrorCode,uss.LastErrorCode, count(uss.LastErrorCode) as 'Count'
from v_r_system sys
 inner join v_gs_operating_system os on os.resourceid=sys.resourceid
 inner join v_updatescanstatus uss on uss.ResourceId=sys.ResourceID
 where uss.lasterrorcode!='0' and os.Caption0 like '%Server%'
Group By uss.LastErrorCode
order by 'Count'
```
Using the results you can then query for the systems that are experiencing the same error
```SQL
select distinct sys.name0 [Computer Name],os.caption0 [OS],convert(nvarchar(26),ws.lasthwscan,100) as [LastHWScan],convert(nvarchar(26),sys.Last_Logon_Timestamp0,100) [Last Loggedon time Stamp],
 sys.user_name0 [Last User Name] ,uss.lasterrorcode,uss.lastscanpackagelocation from v_r_system sys
 inner join v_gs_operating_system os on os.resourceid=sys.resourceid
 inner join v_GS_WORKSTATION_STATUS ws on ws.resourceid=sys.resourceid
 inner join v_updatescanstatus uss on uss.ResourceId=sys.ResourceID
 where uss.lasterrorcode ='-2145107952' and os.Caption0 like '%Server%'
 order by uss.lasterrorcode
```
In this example the error code -2145107952 has a hex value of 0x80244010. Which translates to ![ErrorLookup_80244010]({{site.url}}/media/software-update-troubleshooting-finding-the-problem-children/ErrorLookup_80244010.jpg)

Armed with this info I can begin tacking the largest group of systems with the same error.  While the root cause and resolution can be different depending on the environment these steps will help identify what to focus on.