---
title: "Setting up a global hotkey to mute-unmute a Firefox stream"
date: 2020-03-03T01:50:20Z
draft: false
tags:
    - autohotkey
summary: How to set up a global hotkey to toggle mute a tab of Firefox (for example a Twitch stream) using AutoHotKey
---

## TL;DR

The following AutoHotKey script will let you globally mute-unmute a Firefox tab and return to the window you just had focused.

```
SetTitleMatchMode 2
^!+x::
 WinActivate,Alternate Player
 Send, ^m
 Send, !{tab}
 return
```
Of course you can change the key binding and the text of the name of the tab as needed.

## Long Version

I have a rule that if I spend more than an hour to figure something out, I'll document it *somewhere*, so that my effort can be found via search engine. Usually in the past that's been a reddit thread, but since I have a blog now I'll do this here.

The goal is to have a hotkey I can press any time I'm watching a stream that will toggle mute-unmute the stream, so that I can mute the stream without muting my music during commerical breaks (or any time I want the sound off). I already use [SoundSwitch](https://soundswitch.aaflalo.me/) to toggle my output device, and I've bound `Ctrl+Alt+Shift+Z` to that, so I *could* just press this to kill stream audio, but I'm always listening to music and I don't want to have to lose that sound as well.

If you're not familiar with [AutoHotKey](https://www.autohotkey.com/), it's a scripting language for Windows that lets you bind pretty much anything you could possibly want to any hotkey you could possibly want, in any context you could possibly want. For example, I prefer to use `Alt+[` and `Alt+]` to change tabs, so I have the following rule, which globally overwrites this key combination to the more standard `Ctrl+PageUp/Down`:
```
!]::Send ^{PgDn}
![::Send ^{PgUp}
```

The problem, though, is that I want to send a command to a *non-focused* window. Several years ago, when I first attempted to do this, it was even *more* complicated because the `ctrl+M` hotkey hadn't been enabled by Mozilla yet. I'll go over my first solution first, and explain why it doesn't actually work.

### Attempt 1 - NirCmd

[NirCmd](https://www.nirsoft.net/utils/nircmd.html) is a set of utilities that allow you to do a bunch of useful things from a command line instead of using a UI. One of the commands it provides is `muteappvolume` (documentation [here](https://nircmd.nirsoft.net/muteappvolume.html)). One of the things you can do from the command line, specifically, is:
```
nircmd muteappvolume Firefox.exe 2
```
The `2` means "toggle" - you can also give the argument `1` (mute) or `0` (unmute).

To run this same command from AHK (on the hotkey `Ctrl+Alt+Shift+X`), you can set the following:
```
^!+x::RunWait % "C:\Windows\nircmd.exe" . " muteappvolume firefox.exe 2"
```

But..........this doesn't actually work. It'll show in the volume mixer that it's working, but Firefox has multiple processes, and nircmd doesn't actually mute all of them, the way physically clicking on the mute button in the volume mixer does. So, strike this.

Next I tried:
```
nircmd muteappvolume /8984 2
```

Here `8984` is the process ID (PID) of the specific tab I wanted to mute (found via the task manager). Also no luck.

### Attempt 2 - ControlSend

So eventually I gave up on nircmd. The next thing I wanted to try was `ControlSend`, which lets you send hotkeys to specific programs without actually focusing them. Unfortunately, as of Firefox 73 at least, `ControlSend` just...doesn't work.

The attempt:
```
SetTitleMatchMode 2
^!+x:: ControlSend, MozillaCompositorWindowClass1, ^m,  Alternate Player
```

Not much to say about this, it just, doesn't work. I tried a bunch of iterations of this (along with some help from the AHK Discord support server), and the conclusion seems to be Firefox just straight-up doesn't allow for using `ControlSend` with it.

### Final Attempt - Literally just brute-force hack

So if I can't send this without focusing the window, I guess...I'll just focus the window. Thus the snippet I pasted at the start of this article:

```
SetTitleMatchMode 2
^!+x::
 WinActivate,Alternate Player
 Send, ^m
 Send, !{tab}
 return
```
* `SetTitleMatchMode 2` allows a given title to match part or all of a window's title
* `^!+X` is the hotkey I want, `ctrl+alt+shift+X` (because it's next to my other volume hotkey, `ctrl+alt+shift+Z`)
* `WinActivate` activates a window
* `Alternate Player` is the name of the extension I use to watch Twitch streams - any string you can guarantee is in your tab will work
* `Send, ^m` sends `ctrl+M` to mute the stream
* `Send, !{tab}` presses `alt+Tab` to return me to the program I was in before
* `return` ends the script

### Limitations
This will only mute the currently-focused tab within a window, so if you want to mute a background YouTube tab or something, you'll need to make it its own window. Also, if you started out with that window focused, the final `Alt+Tab` would send you somewhere you don't want to be. For some programs, alt+tabbing back loses your cursor focus from a textarea, which can be annoying. And finally, if you can't find some consistent substring of your window title, you might have to get creative.

That said, to me this is an ENORMOUS quality-of-life improvement, and hopefully this helps someone else too! (Every solution I found while searching was about either nircmd and how it doesn't work, or about ControlSend and how that doesn't work either; I never actually found a working result other than the one I made myself which is outlined here.)