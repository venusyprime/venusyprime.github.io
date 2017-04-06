---
layout: post
title:  "Quickly Licensing Exchange Online users"
date:   2017-03-31 12:00:00 +0000
categories: powershell
---
So here's something that might be useful for other admins looking to move to Exchange Online - particularly if you're used to an on-premises environment, as I was.

To be fully set up for Exchange Online, a user needs a few things:
- They need a *-UsageLocation* set.
- They need a license assigning.
- If you're operating in a hybrid environment with Exchange 2010/2013, the mailbox needs migrating (this post will not cover that part). If you're not in a hybrid environment, assigning the license is enough to create the mailbox.
- For the user to be able to log in, they need their regional configuration (time zone) setting.
- In our case, I also needed to double check the domain used for the *UserPrincipalName* - a lot of our users were set up with a UPN matching our internal domain, when ideally they should login to Office 365 with the external domain that they're familiar with.

## Prerequisites

The most important thing here is the *Azure Active Directory cmdlets for PowerShell*, downloadable from [here](https://docs.microsoft.com/en-gb/powershell/azure/install-msonlinev1). We will be using the v1 cmdlets for this post, as I have not yet played around with the v2 ones.
(The V1 cmdlets are also referred to as the *Microsoft Online Services* cmdlets - hence the MSOL prefix they use).

This post assumes that you have user accounts synchronized from your local AD instance, because that's the scenario I'm familiar with. It also assumes your other domains have been set up properly in Office 365 and local AD, and that you have purchased the appropriate amount of licenses.

## The UserPrincipalName

This isn't mandatory, but does make it easier for users. The company I work for has several trading names, and an internal domain that does not match any of those trading names. Let's call the internal domain *internal.net*, and the domains we actually trade on as *companyA.com*, *companyB.com*. 

>Side note: please don't visit any of these websites; they are being used as examples in place of real names and I cannot speak to the contents of them.

When users give out their email address, it ends in CompanyA.com or CompanyB.com. We set this up using an SMTP alias, tied to the Company attribute in AD. So using that same attribute, we can rewrite the user's UPN to match the domain they'd expect to see.