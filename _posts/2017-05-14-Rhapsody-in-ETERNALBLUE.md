---
layout: post
title:  "Rhapsody in ETERNALBLUE"
date:   2017-05-14 12:00:00 +0000
categories: powershell security
---
So, by now, I probably don't need to tell you about [the wcry ransomware attack](http://www.bbc.co.uk/news/technology-39913630) currently running rampant through the IT systems of Europe (and possibly/probably beyond). Spread with the help of [MS17-010](https://technet.microsoft.com/en-us/library/security/ms17-010.aspx) - also known as [ETERNALBLUE](https://en.wikipedia.org/w/index.php?title=EternalBlue&oldid=780332147).

In this specific case, I wish to look at what someone in my position at my previous employer could have done. I was helpdesk, not responsible for overall network design or hardening - so there are some changes that would never have been implemented had I suggested them, like enabling Windows Firewall on domain networks - and there are things we had in place that would have helped limit damage, like a fairly robust backup solution for most systems. But still, there are things that I could have done.

## A Little Introduction

First, we must have a little introduction to the exploit - just to establish some grounding.

ETERNALBLUE is an exploit in version 1 of the Server Message Block protocol that provides remote code execution capability.

What's the Server Message Block protocol? (Hereafter referred to as SMB for brevity). It controls how Windows (and compatible clients, such as Samba for Linux hosts) handle file sharing. If you have a shared network drive hosted by Windows Server, you're most likely using SMB. It is enabled by default on all modern (and most not so modern) Windows installations. Version 1 has been around since Windows 3.11, version 2 was introduced in Windows Vista/Server 2008, and version 3 with Windows 8/Server 2012.

Generally, SMB traffic is restricted to within an organization's internal domain. However, once one host is infected with malware taking advantage of ETERNALBLUE (say, with a specially crafted email sent to one of the least technical members of staff), it is then trivial for the infection to spread to the rest of the domain network - particularly if Windows Firewall is completely turned off for the domain as well.

## Hardening What We Can

### 1. Install Updates

The most important piece of advice is the same one everyone else is going to tell you. Critical updates need to be applied as soon as possible. Perhaps not *immediately* immediately, with MS having previously screwed up some (and of course, even the most robust testing cannot take account of every configuration of hardware and software), but as soon as is feasible - because we're seeing a much lower time from something going from a leaked or mostly theoretical exploit to being taken advantage of in the wild. (Particularly with things like maliciously crafted Microsoft Office documents).

If you have a WSUS infrastructure, make good use of it. If you do not have a WSUS infrastructure, wave every single article about the spread of the NHS cyberattack to your superiors until you get one. Set a sane update schedule for your environment and work out a plan for getting the most important patches into your organization as soon as you can.

For this particular case, due to the massive spread, [MS has released patches for XP. 2003, and Windows 8](https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/), which they otherwise no longer support. Machines running these systems (along with Windows Vista, which was in support when the ETERNALBLUE patch was issued, but has now fallen out of support) should install these patches ASAP, but with the understanding that this is a temporary measure; that this specific exploit might be corrected, but these systems remain vulnerable to all sorts of threats (present and future) - and so, if they cannot be removed from the network, they must be as isolated as is humanly possible from every other machine.

There are multiple different ways to query a machine's OS, but a quick one is to query the AD computer objects in your domain:

```powershell
Get-ADComputer -Properties operatingsystem -Filter {operatingsystem -like "*2003*"}
Get-ADComputer -Properties operatingsystem -Filter {operatingsystem -like "*XP*"}
Get-ADComputer -Properties operatingsystem -Filter {operatingsystem -like "*Vista*"}
Get-ADComputer -Properties operatingsystem -Filter {operatingsystem -like "* 8 *"} #the spaces need to be here or else it'll get all Server 2008/R2 clients, as well as all Windows 8.1 clients.
#You could combine all of these into one longer query or use a foreach, I'm just trying to keep things simple in this case.
```

However, particularly in environments such as the UK's NHS (with a lot of embedded systems), this perhaps may not tell the full picture - and of course, many of those embedded systems cannot be updated for one reason or another. And this problem isn't specific to embedded Windows systems either; many Linux embedded systems (which are most likely not on the domain at all) also only implemented SMB1 - and even if these implementations may or may not be vulnerable to ETERNALBLUE, they may very well be vulnerable to something else. Find these hosts using whatever you can and lock down whatever is possible - ensure no internet access and they are cordoned off as much as possible from the rest of the corporate network.

### 2. Remove SMB1 From Your Environment

Like most things designed during the Windows 3.11 era of computing, security was not hugely at the forefront of the design of the SMB1 protocol. SMB1 is unnecessary for any version of Windows past Vista/Server 2008 to communicate with any other version of Windows past Vista/Server 2008. If a machine in your environment does not need to talk to one of the problem cases listed above (and very few machines in your environment should - if it is a requirement of something relatively cheap for a business to replace, such as printers from 15 years ago that are falling apart, again, use this as an opportunity to push for things), it is worth disabling - even if this specific exploit may now be patched, there will be others in future.

There are several different ways to accomplish this:

#### a. Deployment Images

If you have a deployment server, disabling SMB1 in your deployment images is an excellent way to make sure no new installation has it. For clients, this can be accomplished by mounting your custom image using `Mount-WindowsImage`. The exact parameters you'll need for this command will vary depending on your environment, so I'd suggest searching for more resources if you're not sure.

If I have my Windows 8/8.1/10 custom image mounted to C:\Mount\, I can then run these commands to disable SMB1 from any future installs from that image.

```powershell
Get-WindowsOptionalFeature -FeatureName SMB1Protocol -Path C:\Mount | Disable-WindowsOptionalFeature -Path C:\Mount -Force
Dismount-WindowsImage -Path C:\Mount -Save
```

This solves the problem for new client deployments - however, most organizations are not redeploying all clients overnight. For "legacy" hosts, the registry setting below must be changed instead - which can be done via modifying the image, or...

#### b. Group Policy (Vista/Win7/Server 2008/Server 2008 R2 Only)

SMB1 is controlled by the following registry key:

![HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters\SMB1](/images/SMB1_Registry_Key.PNG)

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters\SMB1`

Setting this value to 0 disables SMB1. The registry key may still be in use on later versions of Windows, though the [MS documentation](https://support.microsoft.com/en-gb/help/2696547/how-to-enable-and-disable-smbv1,-smbv2,-and-smbv3-in-windows-vista,-windows-server-2008,-windows-7,-windows-server-2008-r2,-windows-8,-and-windows-server-2012) only mentions it being valid for these.

#### c. PowerShell - Against Live Systems (Windows 8+/Server 2012+ Only)

There are three lines of code we need to know; two of which can be easily run *en masse*, and one that is slightly more complicated if you don't have PowerShell Remoting in your environment.

First, `Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force`. This works on both Windows clients and Windows Server. It accepts a `-CimSession` parameter, so it can be run against many machines very quickly by setting up a CIMSession (please see [MS's documentation on New-CIMSession](https://msdn.microsoft.com/en-us/powershell/reference/6/cimcmdlets/new-cimsession) if you're new to CIMSessions).

Next, `Uninstall-WindowsFeature FS-SMB1`. This only works on Windows Server. It accepts a ComputerName parameter so it can be easily run against remote hosts - however, it can only be run against one at once. Use `foreach` and a list of server names to run through things quickly - or use Invoke-Command if remoting is available. Note that the uninstall is only completed after the machine is restarted.

Finally is a variation on the command we saw above: `Get-WindowsOptionalFeature -FeatureName SMB1Protocol -Online | Disable-WindowsOptionalFeature -Online -Force`. This one can only be run on Windows clients (or rather, it can be run on Windows Server, but when I tried it instead of running the command above, the uninstall failed). It supports no remoting capability as part of the cmdlet, so if you have remoting enabled to all clients in your environment, use Invoke-Command, otherwise you're probably looking at creating a startup script.

#### d. PowerShell - DSC

If you're already running DSC in your environment, then you may wish to consider adding something like the below to your configuration:

```powershell
$featuresabsent = @(
    "SMB1Protocol"
    "MicrosoftWindowsPowerShellV2Root" #disable PSv2 to prevent exploits - https://www.youtube.com/watch?v=Hhpi3Sp4W4k
    "MicrosoftWindowsPowerShellV2"
)
foreach ($f_a in $featuresabsent) {
    WindowsOptionalFeature $f_a {
        Name = "$f_a"
        Ensure = "Disable"
    }
}
```

This configuration entry is for clients (in fact, it's a sample for the DSC script I use to configure a new home machine), but it would not be difficult to add for servers - using the WindowsFeature DSC resource instead. If your servers are in push mode, you'd obviously need to push the new config out; if you have a pull server, just wait for them to grab the new information. (Do be aware that this will probably need a restart, so watch out for DSC clients who have auto restart set in their LCM config).

## Wrapping Up

Hopefully, some of this has been of use in helping make your network a little more secure, even if you only have limited access to systems.