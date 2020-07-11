---
title: "Die, DisplayPort!"
date: 2020-07-09T16:28:17Z
draft: false
tags:
    - windows
summary: DisplayPort sucks in a multi-monitor setup and I'm celebrating not using it anymore.
---

When I built my current PC, I was really excited about VR gaming and planned to buy a VR headset and start playing VR games. (Unfortunately, I never got around to getting a VR headset, even three years later....) I invested into a nice graphics card without paying too much attention to what ports it had - DVI, HDMI, and DisplayPort were all probably fine, it had five ports, and I "only" use four monitors, so no problem, right?

Yeah, no. Turns out, Windows has long had an issue with DisplayPort connectors specifically, where, when monitors turn back on after an auto-shut-off, all your windows get rearranged. It occurs when you have a mix of DisplayPort and non-DisplayPort connections in a multi-monitor setup. It's been documented in countless Windows support forum posts with a bunch of useless answers like, "update your graphics driver!" but nothing that actually works. It seems anecdotally like maybe the problem can be solved by having the DisplayPort connection be to your primary monitor rather than non-primary monitors, but really the best solution is just to not have DisplayPort.

When I was using Windows 7, I used a utility called [Persistent Windows](https://www.ninjacrab.com/persistent-windows/) to remember the layout of my windows; it worked relatively well, but I still had failures about once a month or so.

Upon switching to Windows 10 (I was very late to do this, I ran Windows 7 until February of this year), I found that Persistent Windows didn't really work anymore, and I switched to [lapo-luchini's fork](https://github.com/lapo-luchini/WindowsLayoutSnapshot) of [WindowsLayoutSnapshot](https://github.com/adamsmith/WindowsLayoutSnapshot) - the fork fixes a bug in the original with vertical start menus, which is a problem for me because...I use a vertical start menu (it's EXTREMELY nice if you have a multi-monitor setup, I highly recommend giving it a try!).

This utility also has some limitations, though - if you open a new window within a short time of going afk, its position won't be remembered, and sometimes it just completely fails to record positions. Also the z-indices of windows sometimes get rearranged.

So finally I bought a [USB to HDMI adapter](https://www.amazon.com/gp/product/B07RS34B43) and ditched DisplayPort completely. It has a couple limitations - USB 3.0 is needed (though this isn't really a big deal anymore), and you can use it for at most a 1080p monitor, so I had to do a bit of cable rearranging to get that to work out, since only two of my monitors are 1080p and the other two are bigger. The monitor plugged in through this connector is also a bit slower to turn back on than the others, but it's only by a couple seconds, and windows don't ever rearrange, which is the important part.

Other solutions I had considered:
* Buying a different graphics card lol
* With the option of running two cheaper ones in parallel if I couldn't find one that had four non-DP ports
* HDMI or DVI splitter, but these seem to only mirror, not actually let you split signal
* Other window managers, including [DisplayFusion](https://www.displayfusion.com/), but reviews seemed to suggest its support for fixing this issue didn't actually work (I do use this to manage my wallpapers though)
* Never let my monitors turn off
* Use a screen saver?

So, yay, no more DisplayPort ever again!