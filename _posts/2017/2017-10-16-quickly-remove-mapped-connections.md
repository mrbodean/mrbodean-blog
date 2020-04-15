---
layout: post
title: Quickly remove mapped connections
date: 2017-10-16
tags: ["Powershell"]
---

Just a quick tip to remove mapped connections with PowerShell and NET USE

`(net use).split(' ')'%{If($_ -match "\\"){net use /delete $_}}`

To see your connections

`(net use).split(' ')'%{If($_ -match "\\"){$_}}`
