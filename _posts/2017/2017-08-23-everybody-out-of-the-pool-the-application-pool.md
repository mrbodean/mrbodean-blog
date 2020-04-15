---
layout: post
title: Everybody out of the pool, the Application Pool
date: 2017-08-23
tags: ["ConfigMgr","Powershell","SCCM","SQL"]
---

![]({{site.url}}/{{site.baseurl}}/media/big-pool-party.jpg)

Hi my name is MrBoDean and I need to confess that I am not running a supported version of SCCM. Yes, I am migrating to Current Branch but the majority of my system are still in SCCM 2012 R2 without SP1. The reason why is quite boring and repeated and many companies I am sure, but takes a while to tell. So for the past year it feels like duct tape and bailing wire are all that is keeping the 2012 environment up while we try to upgrade between a string of crisis. It is a shame when the Premier Support engineers are on a 1st name basis with you.

So Monday night I was called at 2AM because OSD builds were failing. It happens and most times a quick review of the log files points you in the right direction. Not so much this time around. The builds were failing to even start, everyone was failing with the error
`Failed to get client identity with error 0x80072efe.`
I have 4 Management Points they can not all be down. They are are up and responding when I test them with
```
http://servername/sms_mp/.sms_aut?mplist
http://servername/sms_mp/.sms_aut?mpcert
```
Ok back the the smsts.log for the client that is failing.  It starts up up fine and even does its initial communication for MPLocation and gets a response
`0 https and 4 http locations are returned from MP http://MP4.mrbodean.net.`

It picks the 1st MP in the list and sends a Client Identity Request. That fails quickly with a timeout error.
`failed to receive response with winhttp; 80072efe`
While it does retry it only submits the request to one MP. But the retry fails with the same error and the build fails before it even starts. Nothing stands out initially and being a little groggy, I go for the old stand by of turning it off and back on again. MP1 was the once getting the timeouts so it gets the reboot. After the reboot we try and again and get the same error. At this point a couple of hours have passed and the overnight OSD builds are canceled and I grab a quick nap and start again 1st thing in the morning.  Well that was the plan until the day crew starts trying to do OSD build and everything everywhere is failing. So I open a critical case with Microsoft. While waiting for the engineer to call I keep looking at logs try to identify what is going on. I RDP into MP1 to check the iis configuration and notice that the system is slow to launch applications. I take a peek at task manager and see that RPC requests were consuming 75% on the available memory. To reset those connections and to get the system responsive quickly, down it went for another reboot. Once it came back up, I took a chance and tried to start a OSD build. This time it worked. So the good news goes out to the field techs. Now I just need to figure out what happened to explain why. Management always needs to know why and what are you doing to not let it happen again. About this time the Microsoft engineer calls and we lower the case to a normal severity. I capture some logs for him and to his credit he quickly finds that MP1 was experiencing returning a 503.2 iis status when the overnight builds were failing. To reduce the risk of this occurring again we set the connection pool limit to 2000 for the application pool "CCM Server Framework Pool" on the management points. I get the task of monitoring to make sure the issue does not return and we agree to touch base the next day. Well I am curious about what lead to this and how long it has been going on. Going back the the past couple of days I see a clear spike in the 503 errors Monday evening starting with a few thousand and ramping up to over 300,000 by Tuesday morning. While I recommend using log parser to analyze the iis logs if you are just looking for a count of a single status code you can get it with powershell. This will give you the count of the 503 status with a subcode of 2. (Just be sure to update the log file name to the date you are checking.
```Powershell
(get-content 'C:\inetpub\logs\LogFiles\W3SVC1\u_ex170823.log''?{$_ -match " 503 2 "}).count
```
While I still have not found out why, at least I know what was causing the timeout error. While that knowledge, I finally get some sleep. Surprised that no one called to wake me up because the issue was occurring again, I manage to get into the office early and start looking at the logs again and see another large spike in the 503 errors. I do a quick test to be sure OSD is working and it is. A quick email to the Microsoft engineer and some more log captures leads to an interesting conversation.

We check to make sure that the clients are using all the management points with this sql query
```SQL
select LastMPServerName, count(*) from CH_ClientSummary group by LastMPServerName
```
And we see that the clients are using all the management points but MP1 and MP4 have about twice as many clients as the other two management points. next we check the number of web connections bot of these servers have with netstat: in a command prompt
`netstat -a -n'find /c "80"`
*Just in case you try and run this command in powershell, you will find that the powershell parser will evaluate the quotes and cause the find command to fail. To run the command in powershell escape the quotes.
```
netstat -a -n'find /c `"80`"
```
This showed the MP1 and MP4 were maintaining around 2000 connections each. With a app pool connection limit of 2000 this means that any delay on processing requests  can quickly lead to the application pool being exceeded and lots of 503 errors will result.  So this time the connection pool limit was set to 5000. But a word of caution before you do this in your environment. When a request is waiting in the queue, by default it must complete within 5 minutes or it is kicked out and the request will have to be retried. Be sure that your servers have the CPU and Memory resources to handle additional load that this may cause.

In SCCM 2012 R2 pre SP1 there is no preferred management point. The preferred management points where added in SP1 and improved in Current Branch to be preferred by Boundary Groups. In 2012 your 1st Management point is the preferred MP until the client location process rotates the MP or the client is unable to communicate with a MP for 40 minutes. In this case MP1 is the initial MP for all OSD builds because it is always 1st in a alphabetical sorted list. MP4 is the default MP for the script used for manual client installs. If my migration to Current Branch was done I would be able to assign management point to boundary groups and better balance out the load.  But until then I am tweaking the connection limit on the application pool to keeps things working. Hopefully you are not in the same boat but if you are maybe this can help.

