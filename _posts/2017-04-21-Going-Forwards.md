---
layout: post
title:  "Going Forwards"
date:   2017-04-21 12:00:00 +0000
categories: personal
---

So, my employment was terminated in lieu of notice last week, and I think I'm still coming to terms with that a bit.

The main thing I want to do here is clean up some of my old PowerShell scripts into a better format. (Also, clean up the CSS formatting on the splash page - I am not a web developer by nature!)

* To delete anything that no longer has any relevance.
* To turn the mostly monolithic modules into a much easier to maintain collection of scripts - breaking out each individual function into its own script file, making finding the one I'm looking for to be much easier.
* With ad-hoc scripts that I've written, figure out if there's anything worth keeping and turn that into a function in the module if so.
* To get everything properly version controlled and on GitHub - my old boss didn't really understand the need for version control, resulting in me bodging something together using scheduled `robocopy`s. The Head of IT did understand the need for version control, but the idea to get me set up on the TFS server came too close to the hammer falling.
* To write tests for all functions using Pester.
* To write help for all functions using PlatyPS.

I want to write at least one post a week, to help keep me focused.

----

My plans otherwise are to look for a temporary position, and get my home lab set back up so I can take the 70-411 somewhere around my birthday. Ideally, I also want to use this as a learning opportunity for using PowerShell DSC - I already have basic domain setup scripts, though they need cleaning up. I'd also like to use a Nano Server trial as a DSC pull server.

The main problem is that I'm severely struggling for local storage at the moment, so getting that expanded is priority one - I want to be very sparing if I end up using Azure VMs for this task (though of course, learning more about Azure wouldn't hurt either - except my wallet, and I need to be sparing with that until I find something else).

So yeah, if you're in the UK, in the Barnsley/Sheffield/Wakefield kind of area and looking for an experienced Service Desk Engineer (or Junior Infrastructure Technician) who's pretty decent at scripting things, let me know.