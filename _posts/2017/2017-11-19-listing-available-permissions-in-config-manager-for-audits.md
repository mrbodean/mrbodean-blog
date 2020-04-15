---
layout: post
title: Listing available permissions in Config Manager for Audits
date: 2017-11-19
tags: ["ConfigMgr","SCCM","SQL"]
---

While I love the new pace of development for Configuration Manager there are times I wish the documentation was updated just as fast. It would make somethings much easier. For example I am just stating the first round of audits on Current Branch. No problem I think I did all that documentation at the start of the migration project. Welllll stuff happens;time passes; things change; all that was done for 1511; we are finishing the project on 1702 with 1706 upgrades in a couple of small environments. So of course the audit reports list several new objects that can have permissions applied and where they have been applied. Next thing you know my calendar fills up with meeting to explain everything.  So to make this easier on me and you, I created a report to list the available permissions. You can download the report from [https://gallery.technet.microsoft.com/ConfigMgr-Available-6aec8017?redir=0](https://gallery.technet.microsoft.com/ConfigMgr-Available-6aec8017?redir=0) or [https://github.com/mrbodean/Technet/blob/master/SSRS/ConfigMgr%20Permissions/RBAC%20Available%20Permissions%20by%20Object%20Type.rdl](https://github.com/mrbodean/Technet/blob/master/SSRS/ConfigMgr%20Permissions/RBAC%20Available%20Permissions%20by%20Object%20Type.rdl)

&nbsp;

![]({{site.url}}/media/AvailablePermsReport.jpg)

The "RBAC Available Permissions by Object Type" report will enumerate all the available Securable Object Types and list the permissions that can be set on each object type.

Permission Type Name = The Object Type Name as it appears in the SQL tables and views

Console Name = The name of the Permission Type as it appears in the Configuration Manager console. If this is blank there are two possible reasons. First it is an internal object that is not presented in the console. Second it is a new permission that needs to be mapped to a Console Name. At the point of the initial publication the objects have been mapped for 1702. Running the report on 1706 shows several new permissions that need to be mapped.

Operation = The friendly name of the permission

Bit Flag = This is the Bit Flag require to do the math to determine if the permission is present. While I will use this value on other reports it is presented here for those that want\need to check the values.

Because of the ever changing environment be sure that you test the report. If I made a mistake mapping a object to the console let me know via technet or twitter and I will update the report.