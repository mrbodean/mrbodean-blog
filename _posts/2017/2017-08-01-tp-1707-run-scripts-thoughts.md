---
layout: post
title: TP 1707 Run Scripts Thoughts
date: 2017-08-01
tags: ["ConfigMgr"]
---

I just finished running kicking the tires on the Configuration Manager Technical Preview 1707 Run Scripts feature. This is a post on my personal thoughts about the functionality and as time moves on and more improvements are made all these thoughts may (and hopefully should) become irrelevant.

This has great potential but also poses several challenges.

*   Currently the results of the script execution are hard to report on for most users. For example, Someone important wants you to run a script and report with systems reported XYZ. To do that you have to query SQL and then parse the results. But if you are not the SCCM Admin you may not have any SQL rights to the database. So building custom reports is in your future until there are some canned ones.
*   Releasing this is a large enterprise is almost a no go until the ability to apply scopes and RBAC goodness is available. I can just imagine explaining this to compliance officers and auditors. Oh the meetings we will have.
*   There seems to be plans to support revisions of scripts but for the moment you have to be perfect or start over. As my wife often reminds me, I am good but not perfect. I need to correct mistakes. As admins we get to update all the other objects if needed, we will need to update scripts.
*   A kill switch. One bad script deployed to everything could suck up resources or do worse on every system. Good review and use of the approval workflow should help but you know someone will do it.