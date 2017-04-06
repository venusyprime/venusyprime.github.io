---
layout: post
title:  "Quickly Licensing Exchange Online users"
date:   2017-03-31 12:00:00 +0000
categories: powershell
---
So here's something that might be useful for other admins going through a migration to Exchange Online - particularly if you're used to an on-premises environment, as I was.

To be fully set up for Exchange Online, a user needs a few things:
- They need a *-UsageLocation* set.
- They need a license assigning.
- If you're operating in a hybrid environment with Exchange 2010/2013, the mailbox needs migrating (this post will not cover that part).
- For the user to be able to log in, they need their regional configuration (time zone) setting.

