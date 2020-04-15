---
layout: post
title: Maintenance Window Troubleshooting Tip
date: 2017-10-23
tags: ["ConfigMgr","SCCM"]
---

I keep looking this up so thought I would clearly document it for myself and the rest of the interweb.

When troubleshooting Configuration Manager Maintenance Windows the runtime number in the ServiceWindowManager.log is in seconds
`OnIsServiceWindowAvailable called with: Runtime:6600, Type:2`
6600/60 = 110 minutes

(6600/60)/60 = 1.83 hours, Or 1 hour and 50 minutes

See https://msdn.microsoft.com/library/jj155419.aspx