---
layout: post
title: Lazy is as Lazy does
date: 2015-10-07
tags: ["Powershell","SCCM"]
---

I was happily working away on a couple of scripts for Configuration Manager today, when a case of the monday's hit. We are prepping to change a little over a 1000 distribution points rate limits. In testing the scripts to set the rate limit, a wmi query that returned some useful info so I tucked it away to come back and take a look at later. After getting the functional work done, I started working on documenting the state before and after the changes. That useful info earlier was the array that stores the rate limit. But in testing the documentation script, the value was null. So after comparing the console to the wmi results and a bit of head scratching, I broke down and started reading the documentation. Turns out that SCCM has lazy properties. http://msdn.microsoft.com/en-us/library/cc143276.aspx
> Some Configuration Manager object properties are relatively inefficient to retrieve. If these properties were retrieved for many instances in a class (as might be done in a query), the response would be considerably delayed. Such properties are considered lazy properties and are not usually retrieved during query operations. However, if these properties are retrieved during a query, they have null or zero values, which might not be the actual value of the property for every instance. Therefore, if you want to get the correct value for lazy properties, you must get each instance individually.
> 
So powershell to the rescue. Run the Query for the bulk info

`$dpStatus = gwmi -Computername $siteserver -namespace "root\sms\site_$sitecode" -query "select * from SMS_SCI_address where AddressType = 'MS_LAN'"`

Then a foreach to drill in

`foreach($dp in $dpStatus)`

Once we get to what we want a Get call will populate the Lazy Properties

`$dp.get()`

see the script at [http://gallery.technet.microsoft.com/Report-on-Distribution-80494b6b](http://gallery.technet.microsoft.com/Report-on-Distribution-80494b6b) 

***2020/04/12*** - adding link to github for script due to Technet Gallery decommissioning 
https://github.com/mrbodean/Technet/blob/master/Powershell/DPRatelimits/Publish-DPRateLimits.ps1
