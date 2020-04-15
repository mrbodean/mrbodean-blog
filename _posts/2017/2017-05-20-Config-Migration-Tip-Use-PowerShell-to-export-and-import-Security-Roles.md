---
layout: post
title: Config Migration Tip - Use PowerShell to export and import Security Roles
date: 2017-05-20
tags: ["ConfigMgr","Powershell","SCCM"]
---

I have been doing a lot of migration prep work and wanted to share a big time saver for moving security roles. You can use PowerShell to export and import security roles. If you have lots of custom roles this is a huge time saver.

To export all of the custom roles

`Get-CMSecurityRole 'Where IsBuiltin -eq $false'select RoleId, RoleName'%{Export-CMSecurityRole -RoleId $_.RoleId -Path "C:\Temp\$($_.RoleName).xml"}`

After you collect all the xml files for the roles and are ready to import them use this

`Get-ChildItem -path c:\temp -Filter "*.xml"'%{Import-CMSecurityRole -Overwrite $false -XmlFileName $_.fullname}`