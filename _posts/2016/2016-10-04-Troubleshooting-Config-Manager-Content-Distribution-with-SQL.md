---
layout: post
title: Troubleshooting Config Manager Content Distribution with SQL
date: 2016-10-04
tags: ["ConfigMgr","SCCM"]
---

I have been spending a good portion of my time troubleshooting Content Distribution issue for my distribution points lately and wanted to share my process. The distmgr.log is the starting point for identifying issues. I deal with a large number of distribution points with very active distributions and deployments and sometimes it is helpful to have a way to cut through the noise and focus on distributions that are having issues. I use the following SQL query to start that process
```sql
select SUBSTRING(ServerNALPath, CHARINDEX('\\', ServerNALPath)+2, (CHARINDEX('"]', ServerNALPath)-1) - (CHARINDEX('\\', ServerNALPath)+2)) AS [Distribution Point],*
from v_PackageStatusDistPointsSumm 
Where State !=0
```
For this query a state of 0 indicates the content distribution to that distribution point was successful.  But just because the state is not 0 does not indicate that there is a issue. This query just gives an indication of what is in progress and should be active in the distribution logs. I created a report that will give me a summary of the distributions in progress and how many distribution points are remaining.

![contentreport]({{site.url}}/media/troubleshooting-config-manager-content-distribution-with-sql/ContentReport.png)

The report shows a normal day for me. If I refresh the report  after a few minutes and the overall count goes down for each package then everything is progressing and I will have a good day. But if the counts do not go down I have some checking to do. I can check the Content Status for packages that I need more details on or run the SQL Query to get more detailed information. From there the details drive the next steps. Re-sending the  content, removing and re-sending, adding disk space, resolving network issues, etc.

Happy Troubleshooting