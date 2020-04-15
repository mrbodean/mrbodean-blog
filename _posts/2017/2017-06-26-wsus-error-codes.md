---
layout: post
title: WSUS Error Codes
date: 2017-06-26
tags: ["ConfigMgr","SCCM","WSUS"]
---

I have found that troubleshooting WSUS is like peeling an Onion. Fix one thing only to find another problem. It is enough to make you cry\scream\drink\etc...

This post is how I approach two common issues.  The error codes below come from the client logs and\or SQL. If you need some help pulling the error codes from SQL see [http://www.mrbodean.net/2017/06/25/software-update-troubleshooting-finding-the-problem-children/](http://www.mrbodean.net/2017/06/25/software-update-troubleshooting-finding-the-problem-children/)

* * *

0x80072EE2 - The operation timed out

This can be caused by anything that impacts communication between  the client and the WSUS server. Here is my list to check before asking the network guys what changed:

*   Ensure that the WSUS IIS application pool is running on the WSUS server the client is communicating with.
*   Check the CPU & Memory Utilization on the WSUS server. High utilization by the WSUS IIS application pool can cause timeouts for the clients. This is also a sign that you may need to do some clean up or reindex the WSUS database, see [http://www.mrbodean.net/2017/06/04/wsus-the-redheaded-step-child-of-configuration-manager/](http://www.mrbodean.net/2017/06/04/wsus-the-redheaded-step-child-of-configuration-manager/)
*   Check the event logs on the WSUS server for WSUS IIS application pool crashes. This is a definite sign that you need to do some clean and reindex the WSUS database. see [http://www.mrbodean.net/2017/06/04/wsus-the-redheaded-step-child-of-configuration-manager/](http://www.mrbodean.net/2017/06/04/wsus-the-redheaded-step-child-of-configuration-manager/)
*   Make sure the WSUS server is up. Yes, I know that this should be 1st. But if you follow directions like me, it is right where it should be.
*   Ensure that the client can communicate to WSUS server over the correct port. Use this url and replace the server name and port to match your environment. http://<yourservernamehere>:8530/ClientWebService/susserverversion.xml

    *   If the xml request fails you may have a new firewall and\or acl blocking communication. Bake some cookies and ask the network team what happened. Withhold the cookies until everything works or they prove it is not the network.

* * *

0x80244010 - Exceeded max server round trips

This is a long standing issue with WSUS, see https://blogs.technet.microsoft.com/sus/2008/09/18/wsus-clients-fail-with-warning-syncserverupdatesinternal-failed-0x80244010/

1st step is to decline unused updates. Make sure you only sync what you are patching and decline what is not being used, see [http://www.mrbodean.net/2017/06/04/wsus-the-redheaded-step-child-of-configuration-manager/ ]({% post_url /2017/2017-06-04-wsus-the-redheaded-step-child-of-configuration-manager %}) (It feels like I am beating a dead horse, but you have no ideal how many times that has been the resolution.)

After doing the clean up you may find that you may need to increase the Max XML per Request. By default the xml response is capped at 5MB and limited to 200 exchanges (round trips) See the Microsoft Blog post above. The sql query will below will allow for an unlimited sized response. (BE AWARE THIS CAN HAVE NEGATIVE IMPACTS! - Your network team may come find you and withhold cookies until you stop saturating all the WAN Links.) You may need to turn this on and off to address issues. If you have a large population of clients on the other side of a slow link and need to frequently enable this, you may need to rethink your design for WSUS or SUP for SCCM.
```SQL
USE SUSDB
GO
UPDATE tbConfigurationC SET MaxXMLPerRequest = 0
```
To return this to the default setting
```SQL
USE SUSDB
GO
UPDATE tbConfigurationC SET MaxXMLPerRequest = 5242880
```