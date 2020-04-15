---
layout: post
title: Configuration Manager TP 1707 - Run Scripts
date: 2017-08-01
tags: ["ConfigMgr","Powershell","SCCM"]
---

I want to talk a bit about the new Run Script feature that was added in 1706. In Technical Preview 1707 it gained the option to add parameters to a script. This has the potential to huge benefit many users of Config Manager and is a great example of SAAS quickly delivering functionality.

Creating a script if very straight forward for this example it is just a Query of WMI for Win32_ComputerSystem![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/CreateScript.jpg)

After the script is created, You must approve it. (There is a hierarchy setting to allow\stop authors to approve their own scripts. This should only be allowed in a test environment. ) After the script has been approved it can be run. To run a script go to a collection with the systems you would like to target. You can run the script against the collection as a whole or individual systems in the collection. (You must show the collection membership to target individual systems. The Run Script  option is not available via the default device view.)

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/RunScript.jpg)

Next select the script to run

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/SelectScripttoRun.jpg)

To view the results of the script execution you will need to use Script Status in the Monitoring view.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/ScriptResult.jpg)

Any output from the script is stored in Script Output. For a good peak at what is going on behind the scenes check out this great write up from the [1706 TP by Tom Degreef](http://www.oscc.be/sccm/configmgr/tp/powershell/posh/TP-1706-Run-a-script/)

Now for the new stuff. Parameters!! Create a new script using the same simple wmi query with a parameter.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/CreateScriptwP.jpg)

If you click next you will be able to set the default value for the parameter.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/CreateScriptwP_edit.jpg)

BUG... errr feature alert... If you click next or back without editing the parameter value the edit button is no longer present.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/CreateScriptwP_Noedit.jpg)

Not to worry you will be able to edit the parameter at run time.

When you run a script with a parameter you get a new dialog that allows you to edit the parameter values.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/RunWEdit.jpg)

If you were not able or choose to not set the value when creating the script, click on the parameter name and click edit.  Be sure the parameter name is highlighted or the edit button will not do anything.  I spent a bit thinking  how silly to no be able to edit a parameter more then once. Rechecking my steps proved that was not the case.

Set the parameter value and let the script run.

![]({{site.url}}/{{site.baseurl}}/media/tp-1707-run-scripts/setparmvalue.jpg)

Hopefully this will get you started with running scripts with parameters