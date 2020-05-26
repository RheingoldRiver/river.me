---
title: "VSCode 1"
date: 2020-05-26T06:23:13Z
draft: false
tags:
    - vscode
    - sublimetext
summary: Last week I installed VSCode for the first time, after using exclusive SublimeText for about two years (and Notepad++ prior to that). I'm going to document my learning curve.
---
## Introduction
Last week, after I published my blog post on disambiguation pop-ups, I started outlining my next post, which is about the framework for pop-ups in general that my disambiguations code was built on top of.[^schedule] I typed a couple section names, pasted in some code, and then I wanted to code-fold a completed section in the middle and start writing around it, both before and after.
[^schedule]: Unless I decide to rearrange at the last minute, that will go out next week on Tuesday; my planned schedule (for now at least) is one wiki-code-related post every other week, with optional in-between posts, such as this one. If/when I reach fewer than 6 pre-planned wiki post topics, I'll switch to monthly.

I had a problem though: SublimeText, for all that it is perfect and wonderful and I love it, absolutely *sucks* at code folding. It is the *worst* at code folding. It's completely inexplicable to me how an editor this amazing in so many ways, can be this utterly disastrously bad at code folding - way worse than Notepad++! Sublime's poor code folding was one of the major contributors to my hanging onto NPP usage for so long - I only switched when I was effectively forced to by the existence of the [Mediawiker plugin](https://river.me/blog/sublime-for-mediawiki/).

I've known that VSCode has significantly better Markdown support than Sublime - Sublime's live preview crashes a bunch - so I decided to install VSCode (finally!). I often tell people, "SublimeText is great! You just have to get used to it, get past the learning curve and then it can do everything you want!" I know the same is true of VSCode, so after my first two days using it were completely miserable and a disaster and I hate everything about this stupid IDE why would ANYONE prefer it over Sublime which is CLEARLY SUPERIOR, I decided to not quit using VSCode immediately and instead document my learning curve. Yay!

I have three goals with this post (and subsequent posts in this series, if they happen):
1. Document to myself how much this sucks, so that next time I can appreciate that learning curves will always exist and try not to get too frustrated about being lost at the start of using a new editor/IDE
2. Provide documentation to both myself and anyone else for things I got stuck on and how to fix them
3. Illustrate point 1 to people who read content and think "wow I suck because I can't do xyz" - don't worry, neither can I!
## Step 1: Set up SSH
### Process
The first thing I did was set up SSH to the server I host my blog on. I installed the [Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) extension pack and added my blog server as a host. As in Sublime, everything happens by pressing Ctrl+Shift+P then picking an option from the dropdown there, unless you have hotkeys configured. The command palette prompted me to type an ssh command, so I typed `ssh my.ip.address` and then pressed Connect, expecting to be prompted for more information. A modal window popped up and asked me what type of server (Windows, Linux, macOS); I said Linux. And then it crashed:
```
Could not establish connection to "my.ip.address". The process tried to write to a nonexistent pipe.
```

So I did a google search, found out I needed to add some more stuff to my config file.

Here's what was already there based on what I'd entered into the command palette :
```
Host ip.address
  HostName ip.address
```

Here's what I added:
```
Host ip.address
  HostName ip.address
  User username
  IdentityFile "path\to\key"
```

So I tried reconnecting again, and it turns out, VSCode doesn't like PuTTYgen keys! So after a bunch of searches like `vscode ssh ppk` I found [this page](https://code.visualstudio.com/docs/remote/troubleshooting#_reusing-a-key-generated-in-puttygen) which informs you PuTTY isn't supported and tells you how to convert your key.

Success!

### Verdict
Okay I have to admit, this is a bit nicer than Sublime's remote support. The [normal Sublime plugin](https://codexns.io/products/sftp_for_sublime) isn't free, and it doesn't support showing files in a sidebar (though tbf I never use this functionality regardless), though it does support PuTTY. Being forced into seeing the sidebar actually made me kinda appreciate having it, though this is super conditional on what I'm doing; for example, for wiki development I'd never use a sidebar. The source control (git) sidebar is kinda nice too, though again I don't think I would've sought this out if it wasn't provided to me automatically. And let's be real, I'm not actually going to use it ever. But it looks nice!

Navigating folders is pretty much the same, use command pallette. So the question is really "how often do I want access to a single folder of stuff on my server?" And given my current workflows, the answer is usually "only when I'm writing Python code" and for that I use PyCharm (and switching to VSCode from PyCharm would be a downgrade). So, slight VSCode preference. But really it's only marginally better.

## Step 2: Set up Markdown
### Process 
After a quick search of something like `vscode markdown`, I downloaded [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one). Yay I now have ability to code fold entire sections! After about 30 seconds of typing, though, I came across an extremely annoying roadblock: Quotation marks in Markdown do not get automatically matched. For single quotes, this makes sense (apostrophes exist), but for double quotes, of COURSE I want these matched. In Sublime I remember setting up some regex or something for making matching work how I wanted (ok if we're being fair, I have no memory of anything, but I'm pretty sure I customized because I wanted `{+}` to expand to `{ + }` when you press a space at the cursor position marked by `+`).

I spent probably 2 hours searching for a way to do this, and learning a bit more about VSCode settings. Like Sublime, everything is "just a JSON," but in Markdown you also have a settings window that gives an actual GUI that you can actually use (wow!). Also, language-specific settings go directly into this settings file, which I guess is okay. Same thing for keyboard shortcuts.

So in the end, I discovered that the problem is that the Markdown syntax definition needs to be modified; in the `language-configuration.json` file, `autoClosingPairs` needs to be updated. Unlike Sublime, where you need to install Package Resource Viewer to edit anything, you can just edit files immediately. By default, Markdown is at `C:\Users\River\AppData\Local\Programs\Microsoft VS Code\resources\app\extensions\markdown-basics` (obviously replace `River` with your account name) (it feels really weird to me that it's not in Roaming, but I guess it's because the entire installation is here, not just some files). Because I'm editing basically an install file, I put it under version control and put my changes in a branch. If you really care, you can find the repo [here](https://github.com/RheingoldRiver/vscode-markdown-syntax/tree/my-changes).

### Verdict
I don't love the idea of directly editing installation files on my 2nd day using a text editor. That said, if I had resigned myself to this course of action immediately, instead of trying really hard to not do this, it would have been relatively easy to do.

There are some things missing still - e.g. in Sublime, asterisks auto-"close" in Markdown precisely as desired, i.e. iff it's not the start of a line (because in that case it's probably creating an `li`). Also, I would expect typing a quote directly before a letter to *not* auto-close, but the change I made always auto-closes. That said you could make the argument that there's no valid use case for typing a `"` directly prior a letter; in this case you should instead first highlight the text to quote, and only then press `"`.

I'm guessing if I keep up the level of VSCode I'm using right now (i.e. literally only for this blog), I'll ignore this for now; if I start using VSCode for more things I'll get annoyed pretty quickly and figure out how to do more advanced pattern matching to insert extra characters in the right contexts.

I was pleasantly surprised by how easy it was to add a matching closed *triple* backtick after typing three "opening" ones for a code block; however, the cursor placement after doing so is not ideal, so I really do need to figure out how to set up patterns/macros for this.

## Other
At some point I installed [Sublime Text Keymap and Settings Importer](https://marketplace.visualstudio.com/items?itemName=ms-vscode.sublime-keybindings). I was underwhelmed by what it imported; very few of my settings actually made the migration. But that was pretty expected.

I also installed about 15 different color schemes at random. Currently I'm using [Dracula Official](https://marketplace.visualstudio.com/items?itemName=dracula-theme.theme-dracula). It's pretty and oceanic, and Markdown doesn't require a large array of colors to have full clarity. Not sure yet how to set one scheme per syntax, but let's not get ahead of ourselves, this is still literally just a markdown editor for me.

And finally, I installed a [spell checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) to check for typos in this post (good thing I did). Currently it doesn't give suggestions on right-clicking a misspelled word, but I'll see if I can fix that later.

## Conclusion
I wrote this post in VSCode and I only wanted to punch a hole in my monitor 2 or 3 times! I will be glad to have both this and Sublime available, depending on what I'm doing. Hopefully you found this somewhat useful or entertaining <s>so you could laugh at my misery</s>.