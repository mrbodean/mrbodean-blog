---
layout: post
title: There are no task sequences available for this computer
date: 2016-09-18
tags: ["ConfigMgr","SCCM"]
---

Occasionally I will see the error message "There are no task sequences available for this computer" in my work Config Manager environment. If after checking the simple things like ensuring the  Task sequence is deployed to a collection and deployed correctly. I am forces to move on to the hard stuff like reading the log files. I had a case like this that was a bit of a stumbling block for a while, but thanks to the fine folks a Microsoft Premier Support we were able to resolve everything.

Some background  info: I have a large test environment that we use to test all installs and OSD Builds before moving them to our production environment. Last Thursday I get a report from a team working on a new OSD task sequence that they stopped seeing all of their task sequences in the selection menu. Then on last Friday another team reported the same issue. So I had them collect smsts.log files and send me the info.  So just for fun our patching team started prepping for the release of the next round of patches and started cleanup and creating new software deployment packages. This along with normal deployment activity ran our secondary site out of space twice. So fast forward to this Thursday and we have managed to get everything else working but no OSD builds will start. They are all failing with the error "No assigned task sequence". When reviewing the smsts.log the error occurs right after making the request for policy assignments
```
Retrieving Policy Assignments:    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Successfully read 0 policy assignments.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Retrieving collection variable policy.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Found 0 collection variables.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Retrieving machine variable policy.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Found 0 machine variables.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Setting collection variables in the task sequencing environment.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Setting machine variables in the task sequencing environment.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
Exiting TSMediaWizardControl::GetPolicy.    TSMBootstrap    9/15/2016 5:30:42 PM    1284 (0x0504)
WelcomePage::OnWizardNext()    TSMBootstrap    9/15/2016 5:30:42 PM    1248 (0x04E0)
Loading Media Variables from "D:\sms\data\variables.dat"    TSMBootstrap    9/15/2016 5:30:42 PM    1248 (0x04E0)
no password for vars file    TSMBootstrap    9/15/2016 5:30:42 PM    1248 (0x04E0)
No assigned task sequence.    TSMBootstrap    9/15/2016 5:30:42 PM    1248 (0x04E0)
Setting wizard error: There are no task sequences available for this computer.    TSMBootstrap    9/15/2016 5:30:42 PM    1248 (0x04E0)
```
I have seen similar behavior with corrupt policy and checked for that with this SQL query

`SELECT * FROM ResPolicyMap WHERE machineid = 0 and PADBID IN (SELECT PADBID FROM PolicyAssignment WHERE BodyHash IS NULL)`

If you get any rows returned you can remove the bad policy records with this SQL statement

`Delete FROM ResPolicyMap WHERE machineid = 0 and PADBID IN (SELECT PADBID FROM PolicyAssignment WHERE BodyHash IS NULL)`

But in this case there were no rows returned so in lieu pulling the remaining two strands of hair off my head , A case was opened with premier support. After several hours of checking and rechecking settings and trying various things we where still in the same boat, then the support engineer said try this SQL query

`SELECT * FROM MachineIdGroupXRef`

While checking the return for the row with our machineID, we noticed that the ArchitectureKey value was not set correctly. In this case the ArchitectureKey  value was the negative value of the machineID. Because we were working with the unknown computer records we set ArchitectureKey  for those entries to 2. If you are working with an existing client record the correct value would be 5.  After correcting those values the OSD build began to pull policy assignments normally.

*** 10/4/16 Updated SQL statements to only Identify unknown computer records. Deletes of regular computer records will not cause the issue and the fix will partially restore the deleted object.

If you are in the same boat use this SQL query to identify the issue

`SELECT * FROM MachineIdGroupXRef where ArchitectureKey Not In (2) AND GroupKey = 0`

And this SQL Statement to correct the issue

`Update MachineIdGroupXRef Set ArchitectureKey = 2 Where GroupKey = 0 And ArchitectureKey <> 2`

We are still doing some post issue research for root cause but it appears that a admin user deleted all members of a collection rather then removing the membership rule on the collection. This appeared to cause the  negative values and opened the door to OSD Hell week.
