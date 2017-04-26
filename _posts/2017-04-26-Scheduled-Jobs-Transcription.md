---
layout: post
title:  "Scheduled Job: PowerShell Transcription Logging and OneDrive"
date:   2017-04-26 12:00:00 +0000
categories: powershell
---
Since the option became available, I've been using PowerShell transcription logging on my personal devices by setting a local Group Policy. (For more information on it and the other PowerShell related Group Policies, please see [this article](https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/) - particularly if you're in a domain environment and need to make sure no malicious commands are being run). It's really useful for being able to go back and search through the commands that I've run on any machine (as is Ctrl+R for local history searching).

However, there's a problem. I had my transcript folder set up as a folder within OneDrive. Since I tend to leave PowerShell sessions open for a long time rather than closing and reopening them, this meant that OneDrive was constantly trying to process a file that it couldn't touch (because the transcript file is only available to upload after the session it relates to is closed). On my desktop, the impact of this was barely noticeable, but on my laptop, it was causing OneDrive to use a lot of CPU that it didn't need to, and drain my battery a lot faster than it should have done.

So, a relatively easy fix then:

1. `mkdir C:\PowerShellTranscripts\`
2. Change the Group Policy to point to C:\PowerShellTranscripts\ rather than previous location in OneDrive.
3. Create a new [scheduled job](https://msdn.microsoft.com/en-us/powershell/reference/5.1/psscheduledjob/about/about_scheduled_jobs) to move files from C:\PowerShellTranscripts to the location I want them in in OneDrive.

``` powershell
$script = {
    $TranscriptsPath = "C:\PowerShellTranscripts"
    $OutputPath = Join-Path -Path $env:OneDrive -ChildPath "Documents\PowerShell\PowerShell Transcripts"
    $DaysDelay = 3

    Get-ChildItem $TranscriptsPath | where LastWriteTime -lt (get-date).AddDays(-$DaysDelay) | Move-Item -Destination $OutputPath
}
$option = New-ScheduledJobOption -StartIfOnBattery
$trigger = New-JobTrigger -Daily -At "8PM"
Register-ScheduledJob -Name "Migrate Transcripts" -ScriptBlock $script -MaxResultCount 14 -ScheduledJobOption $OPTION -Trigger $trigger
```

I'm not sure when the `$env:OneDrive` variable was added, but it saves me having to look it up in the registry. (If you're not on Windows 10, you may need to get this location by running `Get-ItemProperty HKCU:\Software\Microsoft\OneDrive\ | select -ExpandProperty UserFolder`).

This is a lot easier for OneDrive to deal with, as it's getting a few KB of files each day at 20:00, rather than having to constantly process.

Going forward, I'll probably add the C:\PowerShellTranscripts folder and configuring the registry key for transcription logging as part of the scripts I use when setting up a new personal machine. (In an environment where the machine likely has more than one user, you would want to make sure that the C:\PowerShellTranscripts folder is secured as well).