---
layout: post
title: Why you should not like "like"
date: 2016-10-06
tags: ["ConfigMgr","SCCM","WQL"]
---

So for the past two days I have been checking and triple checking the Configuration Manager environment I support at work. Nothing like a hurricane being labeled "worst case" storm to make you shake the dust of the DR plans. After all the backup checks and distribution point health and content validations, I started looking and the performance of various components. Overall nothing major found but while checking the collection evaluations I did find a few collections that stood out for poor performance. Only one was real nasty and over 2 minutes. The collection is not very large in terms of members but the query populating it needed a little work. So before I dig into the details, you may like to know how to identify the issue. You can find all the info you need in the "colleval.log".  If you use a little googlefu there are some good tips on how to parse the log with powershell and identify your troublemakers. Or you can use the Collection Evaluation Viewer from the System Center 2012 R2 Toolkit. If you have never used it before [The Config Ninja has a great post ](https://blogs.technet.microsoft.com/smartinez/2015/07/31/collection-evaluation-one-tool-one-report/)walking you through it and some reports to display the same info.

With the Collection Evaluation Viewer you can use the run time to identify collections that need some review. When you identify a collection to review open the properties and look at the membership rules.  Here is an example of a collection query that was running longer then it should.
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client 
from SMS_R_System inner join SMS_G_System_SoftwareProduct on SMS_G_System_SoftwareProduct.ResourceID = SMS_R_System.ResourceId 
where SMS_R_System.ADSiteName = "SOMEADSITE" And SMS_G_System_SoftwareProduct.ProductName like "%SAP GUI for Windows%"
```
In some environments this may complete in just a few seconds but it was taking over 2 minutes for me. On the whole this is a fairly normal requirement for a deployment. All computers in a location with Software X installed. But the database has to get all of the system records in the location and then check all of the product names it has reported and check to see if the name is "Like" the value in the query. Now there is nothing wrong with Like and there are lots of cases where you must use it. But you have to understand that it is a more expensive cost to SQL queries using them. So to check out what the possible returns were and what the wild cards were allowing the query to collect I queried the view in sql.
```SQL
select v_GS_SoftwareProduct.ProductName 
from v_GS_SoftwareProduct
where v_GS_SoftwareProduct.ProductName Like '%SAP GUI for Windows%'
Group BY v_GS_SoftwareProduct.ProductName
```
And the query returned a single product name of 
`SAP GUI for Windows`

So to solve this query's performance issue, I just switched to = and the collection evaluation run time went down to 3.5 seconds
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client from SMS_R_System inner join SMS_G_System_SoftwareProduct on SMS_G_System_SoftwareProduct.ResourceID = SMS_R_System.ResourceId where SMS_R_System.ADSiteName = "SOMEADSITE" and SMS_G_System_SoftwareProduct.ProductName = "SAP GUI for Windows"
```

Now that the longest running evaluation was resolved there where a couple of collections that where taking 15 - 20 seconds to complete. Not terrible but not good either, as I looked through them I found a couple of things to share. First up is another collection for computers with a specific type of software installed.
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client 
from SMS_R_System inner join SMS_G_System_ADD_REMOVE_PROGRAMS on SMS_G_System_ADD_REMOVE_PROGRAMS.ResourceID = SMS_R_System.ResourceId 
where SMS_G_System_ADD_REMOVE_PROGRAMS.DisplayName LIKE "ODBC Driver for Teradata 14"
```
So I go to SQL and check how many Display names are returned by the wild card query and get back two. So this time changing to a "in list" query reduced collection evaluation time.
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client 
from SMS_R_System inner join SMS_G_System_ADD_REMOVE_PROGRAMS on SMS_G_System_ADD_REMOVE_PROGRAMS.ResourceID = SMS_R_System.ResourceId 
where SMS_G_System_ADD_REMOVE_PROGRAMS.DisplayName in ("ODBC Driver for Teradata 14.10","ODBC Driver for Teradata 14.10.0.2")
```
One thing to make note of is you need to consider is if the values being returned are going to change often and will you know about the changes. But in general I would use the original query with an ad-hoc query or a report. The explicit values for the collection membership query are appropriate because of the impact to the collection evaluation process.

For the last example I am going to use a query that need to use like.  This query is evaluating the computer name and the author needed to include systems with a specific range of ending values and a specific character starting the computer name. Along with a few exclusions.
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client 
from SMS_R_System   
where (SMS_R_System.Name like "X%%%%A101" or SMS_R_System.Name like "X%%%%A102" or
SMS_R_System.Name like "X%%%%A103" or SMS_R_System.Name like "X%%%%A104" or 
SMS_R_System.Name like "X%%%%A105" or SMS_R_System.Name like "X%%%%A106" or 
SMS_R_System.Name like "X%%%%A107" or SMS_R_System.Name like "X%%%%A108" or 
SMS_R_System.Name like "X%%%%A109" or SMS_R_System.Name like "X%%%%A110" or 
SMS_R_System.Name like "X%%%%A111" or SMS_R_System.Name like "X%%%%A112" or 
SMS_R_System.Name like "X%%%%A113" or SMS_R_System.Name like "X%%%%A114" or 
SMS_R_System.Name like "X%%%%A115" or SMS_R_System.Name like "X%%%%A116" or 
SMS_R_System.Name like "X%%%%A117" or SMS_R_System.Name like "X%%%%A118" or 
SMS_R_System.Name like "X%%%%A119" or SMS_R_System.Name like "X%%%%A120" or 
SMS_R_System.Name like "X%%%%A121" or SMS_R_System.Name like "X%%%%A122" or 
SMS_R_System.Name like "X%%%%A123" or SMS_R_System.Name like "X%%%%A124" or 
SMS_R_System.Name like "X%%%%A125" or SMS_R_System.Name like "X%%%%A126" or 
SMS_R_System.Name like "X%%%%A127" or SMS_R_System.Name like "X%%%%A128" or 
SMS_R_System.Name like "X%%%%A129" or SMS_R_System.Name like "X%%%%A130" or 
SMS_R_System.Name like "X%%%%A131" or SMS_R_System.Name like "X%%%%A132" or 
SMS_R_System.Name like "X%%%%A133" or SMS_R_System.Name like "X%%%%A134" or 
SMS_R_System.Name like "X%%%%A135" or SMS_R_System.Name like "X%%%%A136" or 
SMS_R_System.Name like "X%%%%A137" or SMS_R_System.Name like "X%%%%A138" or 
SMS_R_System.Name like "X%%%%A139" or SMS_R_System.Name like "X%%%%A140" or 
SMS_R_System.Name like "X%%%%A141" or SMS_R_System.Name like "X%%%%A142" or 
SMS_R_System.Name like "X%%%%A143" or SMS_R_System.Name like "X%%%%A144" or 
SMS_R_System.Name like "X%%%%A145" or SMS_R_System.Name like "X%%%%A146" or 
SMS_R_System.Name like "X%%%%A147" or SMS_R_System.Name like "X%%%%A148" or 
SMS_R_System.Name like "X%%%%A149" or SMS_R_System.Name like "X%%%%A150" or 
SMS_R_System.Name like "X%%%%A151" or SMS_R_System.Name like "X%%%%A152" or 
SMS_R_System.Name like "X%%%%A153" or SMS_R_System.Name like "X%%%%A154" or 
SMS_R_System.Name like "X%%%%A155" or SMS_R_System.Name like "X%%%%A156" or 
SMS_R_System.Name like "X%%%%A157" or SMS_R_System.Name like "X%%%%A158" or 
SMS_R_System.Name like "X%%%%A159" or SMS_R_System.Name like "X%%%%A160" or 
SMS_R_System.Name like "X%%%%A161" or SMS_R_System.Name like "X%%%%A162" or 
SMS_R_System.Name like "X%%%%A163" or SMS_R_System.Name like "X%%%%A164" or 
SMS_R_System.Name like "X%%%%A165" or SMS_R_System.Name like "X%%%%A166" or 
SMS_R_System.Name like "X%%%%A167" or SMS_R_System.Name like "X%%%%A168" or 
SMS_R_System.Name like "X%%%%A169" or SMS_R_System.Name like "X%%%%A170" or 
SMS_R_System.Name like "X%%%%A171" or SMS_R_System.Name like "X%%%%A172" or 
SMS_R_System.Name like "X%%%%A173" or SMS_R_System.Name like "X%%%%A174" or 
SMS_R_System.Name like "X%%%%A175" or SMS_R_System.Name like "X%%%%A176" or 
SMS_R_System.Name like "X%%%%A177" or SMS_R_System.Name like "X%%%%A178" or 
SMS_R_System.Name like "X%%%%A179" or SMS_R_System.Name like "X%%%%A180" or 
SMS_R_System.Name like "X%%%%A181" or SMS_R_System.Name like "X%%%%A182" or 
SMS_R_System.Name like "X%%%%A183" or SMS_R_System.Name like "X%%%%A184" or 
SMS_R_System.Name like "X%%%%A185" or SMS_R_System.Name like "X%%%%A186" or 
SMS_R_System.Name like "X%%%%A187" or SMS_R_System.Name like "X%%%%A188" or 
SMS_R_System.Name like "X%%%%A189" or SMS_R_System.Name like "X%%%%3190" or 
SMS_R_System.Name like "X%%%%3191" or SMS_R_System.Name like "X%%%%3192" or 
SMS_R_System.Name like "X%%%%3193" or SMS_R_System.Name like "X%%%%3194" or 
SMS_R_System.Name like "X%%%%3195" or SMS_R_System.Name like "X%%%%3196" or 
SMS_R_System.Name like "X%%%%3197" or SMS_R_System.Name like "X%%%%3198" or 
SMS_R_System.Name like "X%%%%3199") AND SMS_R_System.Name not like "XB9%" AND 
SMS_R_System.Name not like "XC9%"
```
Right away we know the original author did not understand [WQL operators](https://msdn.microsoft.com/en-us/library/aa392263(v=vs.85).aspx).  By using the correct operators when you must use like, the query is simplified and performs much better.
```SQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client 
from SMS_R_System 
where (SMS_R_System.Name like "X____A1[0-9][0-9]" 
or SMS_R_System.Name like "X____319[0-9]") 
AND SMS_R_System.Name not like "XB9%" 
AND SMS_R_System.Name not like "XC9%" 
AND SMS_R_System.Name not like "X____A100"
```
Evaluating for single characters with an underscore "_" is quicker then using the percent "%" for any and all character combinations. If you need to query for a specific number of any characters use multiple underscores. Specifying the range allows the query to be much shorter and simpler.
7