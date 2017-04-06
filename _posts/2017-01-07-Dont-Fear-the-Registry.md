---
layout: post
title:  "Don't Fear the Registry"
date:   2017-01-07 12:00:00 +0000
categories: windows powershell
---

I'm not a fan of some of the advice about the registry you see online. You know the type.

> ⚠ You should only view the registry exactly as described in this guide! Your computer may explode if you even as much as think about opening REGEDIT without clear direction!

I mean, you should not go in and modify things willy-nilly, that much is common sense. But this level of warning discourages even the simplest exploration into it - to stop people even looking for solutions to problems in there without being told.

I follow the "look, don't touch (usually)" principle - after all, no registry entry should change simply by observing them in REGEDIT or PowerShell (and any that do change on observation have a decent chance of being malware).

I use OneDrive for sharing files between my desktop and my Surface 3, and editing things like this from anywhere. My OneDrive folder on my desktop is on my large media drive (M:\OneDrive), while on my Surface, it's in the default location of `~\OneDrive` - as OneDrive can only be placed on an NTFS drive, and the micro SD card in the Surface 3 is FAT.

If I need to use a file path in a script that is intended for personal use only, I don't know which device I'm on. I don't particularly like relative paths in scripts if the relative path isn't in the same folder as the script (or a subdirectory). I don't want to create a fake M:\ drive on my Surface, or create a symlink from ~\Desktop on my desktop.

OneDrive folders are user specific, so I know that I'm looking in `HKEY_CURRENT_USER`. PowerShell makes it pretty easy to locate the registry entry I'm looking for:

``` PowerShell
Set-Location HKCU:\
Get-ChildItem *OneDrive* -Recurse

Hive: HKEY_CURRENT_USER\SOFTWARE\Microsoft
Name                           Property
----                           --------
OneDrive                       EnableDownlevelInstallOnBluePlus : 0
                                # output snipped
                                UserFolder                       : C:\Users\Charles\OneDrive
```

So now I know I have a consistent place to look for where the OneDrive folder actually is. I can wrap a helper function around that, put that in a module, and boom, all I need to do is call `Get-OneDrivePath` when I want to know where to put something in PowerShell - no matter which machine I'm using.