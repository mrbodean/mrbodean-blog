---
layout: post
title: Software Updates not applying after a Site Reset
date: 2017-05-10
tags: ["ConfigMgr","SCCM"]
---

There is a long story about why that I have not had time to post about, one of the SCCM environments had to be recovered with a Site Reset. The reset was successful and everything appeared to be functioning normally. The next day the patching team started to report a few clients not applying patches. Now this is not unusual, there are always some clients that have issues, but by the end of the day it they were reporting that it was all clients.  The bulk of the clients were reporting  **'Assignment ({xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}) already in progress state (AssignmentStateDetecting). No need to evaluate UpdatesDeploymentAgent' **in the UpdateDeployment.log. [Gabriel Alicea](https://www.linkedin.com/in/gabriel-alicea-mcts-87661510) has a great post that solved the issue - [https://www.linkedin.com/pulse/total-actionable-updates-0-gabriel-alicea-mcts](https://www.linkedin.com/pulse/total-actionable-updates-0-gabriel-alicea-mcts).

The moral of the story is that the Site Reset changed the version of the wsus catalog to 1 on the primary server but the software update point and the database had a different number. Stopping the services on the primary,updating the registry values, restarting the services and then running a sync allowed the clients to correctly evaluate the scans and apply patches.

&nbsp;