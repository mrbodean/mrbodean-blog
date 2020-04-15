---
layout: post
title: Management Point Troubleshooting
date: 2017-03-07
tags: ["ConfigMgr","SCCM","SQL"]
---

Just a quick note, If you are looking into issues with a management point responding with a 500 error for policy.

If `http://servername/SMS_MP/.SMS_AUT?MPCERT` is ok

and `http://servername/SMS_MP/.SMS_AUT?MPLIST` gives a 500 error.

Check your database it could be down. Not how I wanted to end a Monday but at least in this case it was a SAN network issue impacting the database server. Once that was resolved the SCCM site came right back up.