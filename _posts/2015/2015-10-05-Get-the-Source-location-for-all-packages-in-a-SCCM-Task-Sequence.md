---
layout: post
title: Get the source location for all packages in a SCCM Task Sequence
date: 2015-10-05
tags: ["Powershell"]
---

The past year or so I have been working on improving the processes we use at work to manage SCCM. One of the painful task we have is verifying that all of the package content used in a task sequence is being pulled from a source control deployment location. We average about 30 packages per OSD task sequences and after clicking on the properties of about the third package in the console, I had find a better way. 

{% gist 4ece5b533903ff9d9400 %}


Now any time I have to do the validation 
```
Get-TSPkgSource -PackageId "DEV005A4" -SiteCode "DEV" ' out-gridview
```

Powershell does the heavy lifting and I can grab some coffee. 