---
title: "A new-tab-in-container Firefox hotkey"
date: 2020-07-01T15:09:08Z
draft: false
tags:
    - autohotkey
    - firefox
summary: A new AutoHotKey mini-adventure, in which I want a hotkey to create a new tab in a specific container, but here there be dragons!
---

Recently, Fandom introduced a policy that all Google Analytics access must be done via @fandom.com email addresses, which required something to change for me since my primary email that I'm always logged into is @gmail.com. Naturally, I was determined to adapt my tools to my workflow rather than my workflow to my tools. This was surprisingly complicated, so I'll document what I did.

## Requirements
1. Pressing Ctrl+Y needs to create a new tab in a "Work" container.
2. I do not want to add any buttons to my Firefox UI.
3. Ctrl+Shift+2 is a global hotkey for [ShareX](https://getsharex.com/) and needs to remain that way.

## Tools
* [Firefox Multi-Account Containers](https://addons.mozilla.org/en-US/firefox/addon/multi-account-containers/)
* [Easy Container Shortcuts](https://addons.mozilla.org/en-US/firefox/addon/easy-container-shortcuts/?src=search)
* [AutoHotKey](https://www.autohotkey.com/)

## Method
Note: All of these scripts are inside of an `#IfWinActive Firefox` (my `Maps.ahk` file starts with `SetTitleMatchMode 2`).
### The first AHK script
If I compromise on the second requirement and allow a UI button in Firefox, the following script works:
```
^y::
	Send ^. ; ctrl+. brings up the context menu of the add-on button
	Sleep 300 ; wait for it to load
	Send {Tab} ; and then you can navigate through the buttons
	Send {Tab} ; `Work` is the 2nd profile
	Send {Enter}
	Return
```
Then I learned you can access a profile via its number:
```
^y::
	Send ^.
	Sleep 300
	Send 2
	Return
```
Problems: The second requirement isn't satisfied; this requires there to be a button added to my Firefox UI, which I'm unwilling to allow, because I have only a single row of UI space that includes navigation, drop-down menus, the URL bar, all my add-ons buttons, and all of my one-click bookmark icons.

### The second AHK "script"
So after some searching I found the add-on "Easy Container Shortcuts" linked above. The problem with this is that these shortcuts overlap with my ShareX shortcuts, as mentioned in requirement 3. First I decided to see if ShareX intercepts keystrokes before or after AHK. So I tried this:
```
^y::
	Send ^+2
```

Turns out, ShareX comes after AHK, and this fails.

My next thought was to fork the extension, since the hotkeys aren't configurable. But then I remembered....

### The third AHK script
ControlSend! ControlSend is [what I used before when I set up a global stream mute-unmute hotkey](https://river.me/blog/global-hotkey-mute-firefox-stream/).
```
^y::
	IfWinActive ahk_class MozillaWindowClass
	ControlSend,ahk_parent, ^+2
```
This can't possibly be intercepted by ShareX because it's sent directly to Firefox! Still, there's one problem - it doesn't work.

(What's up with `IfWinActive ahk_class MozillaWindowClass`? See my [previous blog post](https://river.me/blog/global-hotkey-mute-firefox-stream/) about the mute-unmute hotkey and sending to Firefox)

### The final AHK script
After a quick Google search, I [found an explanation](https://www.autohotkey.com/boards/viewtopic.php?f=14&t=34381) - ControlSend doesn't handle modifiers the way you might want them to when they interfere with the active window. As this is intended behavior, I fixed this by using the intended method instead:

```
^y::
	IfWinActive ahk_class MozillaWindowClass
	ControlSend,ahk_parent, {Ctrl down}{Shift down}2{Shift up}{Ctrl up}
```

And ta-da! We are done.