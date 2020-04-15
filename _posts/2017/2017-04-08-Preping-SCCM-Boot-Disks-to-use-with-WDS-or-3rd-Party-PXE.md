---
layout: post
title: Preping SCCM Boot Disks to use with WDS or 3rd Party PXE
date: 2017-04-08
tags: ["ConfigMgr","Powershell","SCCM"]
---

I have been busy after taking an extended vacation and then catching up at work, but a couple of folks on twitter have been sharing about using the SCCM Boot Disks with WDS. This is something that I do and learned from [Johan Arwidmark](http://deploymentresearch.com) . If you work with Configuration Manager you may have heard of him before. :) Here his current post on the subject [http://deploymentresearch.com/Research/Post/612/Making-a-ConfigMgr-boot-image-work-with-standalone-WDS-or-3rd-party-PXE-server](http://deploymentresearch.com/Research/Post/612/Making-a-ConfigMgr-boot-image-work-with-standalone-WDS-or-3rd-party-PXE-server) and it has a link to [Zeng Yinghua (Sandy)](http://www.thesccm.com/ipxe-sccm/)<span style="color: #007600;">'s post on this using iPXE rather then WDS.</span>

What I have to add is a quick Powershell script to automate prepping the Boot Disk for use outside of a integrated SCCM PXE DP.

Step 1 - Create the Boot disk as a ISO [https://docs.microsoft.com/en-us/sccm/osd/deploy-use/create-bootable-media](https://docs.microsoft.com/en-us/sccm/osd/deploy-use/create-bootable-media)

Step 2 - Extract the ISO contents.

Because I do this for a couple of boot disk in several environments I name the iso and the directory that they are extracted to something that helps keep track of them easily. For example Lab_x64, Lab_x86, Prod_x64, etc ..

The script below will use the folder name that you extract the files to name the wim file as well. While not a big deal if you only do one disk at a time, it helps when processing several at the same time.

Step 3 - Prep the boot.wim file for use via PXE.

This is the script that I use
```Powershell
$sources = "D:\SCCM Export\Lab_x64","D:\SCCM Export\Lab_x86","D:\SCCM Export\Prod_x86","D:\SCCM Export\Prod_x64"
$mountdir = "D:\Work\mount"
$outputdir = "D:\SCCM Export\tocopy"

Foreach($source in $sources){
    dism /mount-wim /wimfile:"$source\sources\boot.wim" /index:1 /mountdir:"$mountdir"
    If(!(Test-Path "$mountdir\SMS\Data")){new-item -ItemType Directory -path "$mountdir\SMS\Data"}
    $files = get-Childitem -path "$source\SMS\Data"
    $files'Copy-Item -Destination "$mountdir\SMS\Data" -force -Verbose
    remove-variable files
    If(!(Test-Path "$mountdir\Tools")){new-item -ItemType Directory -path "$mountdir\Publix"}
    $files = get-Childitem -path "D:\SCCM Export\WDS\Tools"
    $files'Copy-Item -Destination "$mountdir\Publix" -force -Verbose
    remove-variable files
    copy-item -path "D:\SCCM Export\WDS\TSconfig.ini" -destination "$mountdir" -force -Verbose
    dism /unmount-wim /mountdir:"$mountdir" /commit
}

Foreach($source in $sources){
 Copy-Item -Path "$source\sources\boot.wim" -Destination "$outputdir\boot_$(Split-Path $source -leaf).wim" -Force -Verbose
}
```
This will go thought each of the directories in the $sources variable and mount the boot wim with dism. If there is not a \SMS\Data  folder it will create it. Next it copies the contents of the SMS\DATA folder from the source directory extracted from the iso and copies it to the mounted wim file. After that the script does an optional step. For my environment when we use WDS there is need to execute a Pre-Execution step. The files for this are staged in the Tools directory. So the script creates the directory and copies the files. Next it copies the TSconfig.ini for the Pre-Execution step. This is also optional.  The script then unmounts the wim file and commits the changes.

After all of the boot.wim files have been updated the script will copy each to a staging directory and name the files based on the name of the source folder. So Lab_x64.iso was extracted to d:\SCCM_export\Lab_x64 and the boot.wim from that directory is named Lab_x64.wim.

Step 4 - Copy to your WDS server and add to the menu.

Thanks Johan and Sandy for posting and reminding me about this.

&nbsp;