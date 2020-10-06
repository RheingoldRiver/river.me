---
title: "Run Pywikibot in Windows Subsystem for Linux"
date: 2020-09-22T07:48:29Z
draft: false
tags:
    - pywikibot
    - windows
    - linux
summary: How to configure PWB for third-party wikis, specifically in Windows Subsystem for Linux
description: How to configure PWB for third-party wikis, specifically in Windows Subsystem for Linux
---
[Pywikibot](https://www.mediawiki.org/wiki/Manual:Pywikibot) is a set of command-line tools written in Python that can be invaluable when performing administration tasks on wikis. It's made even better by running it in a shell that doesn't suck, which unfortunately does not include the Windows command prompt.

In this post I'll first go over how to install and configure PWB for third-party (non-WMF) wikis (I actually reinstalled/updated PWB for the first time in three years just to write this), and then I'll show you how to set up [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) with the [Fish shell](https://fishshell.com/). There's a number of QOL improvements over the standard Windows command prompt, including it:
* Has color highlighting including things like matched/unmatched quotes
* Remembers your command history and suggests autocompletions
* Allows you to manage and clear said history as needed if you have unwanted suggestions
* Doesn't lag for like 10 seconds when you press ctrl+C in the middle of a batch job omg
* Lets you set aliases incredibly easily, which work from any directory, without adding anything to PATH

As a warning, if you aren't comfortable with command-line operations, you'll probably have a very uncomfortable time with this and hate every minute of it. I assume you're familiar with commands like `cd` and `ls` already, and with the general concept of typing things into a terminal and having things happen, since that's how PWB works. If not, and you don't want to use this tool, that's okay! Check out [AutoWikiBrowser](https://en.wikipedia.org/wiki/Wikipedia:AutoWikiBrowser) instead, a GUI-based tool that can accomplish some of the same things PWB can (though it is admittedly significantly more limited in scope). And then maybe come back to this later!

Also, you will have to edit some of the source code of PWB to workaround a couple things where it doesn't play nice with WSL. Sorry.

While I did redo my entire PWB installation as I was writing this post, it's always possible I left out a minor but important step or made a typo or something, so please [let me know](https://twitter.com/RheingoldRiver) if anything doesn't work as advertised - I'll help you troubleshoot and update as needed!
## Pywikibot
### Getting PWB
You can follow instructions [here](https://www.mediawiki.org/wiki/Manual:Pywikibot/Installation). However I will also provide my own instructions. (Note that if you're reading this post more than 6-ish months after publication, my instructions may be outdated, so if something doesn't work keep that in mind.)

#### Getting Python
To install Python in Windows, my preferred method is first to install the package manager [Chocolatey](https://chocolatey.org/install) and then run `choco install python`. This will take care of all of the PATH shit for you and also give you `pip` and stuff without you having to worry.

There are certainly other package managers available for Windows, including [an official one](https://docs.microsoft.com/en-us/windows/package-manager/winget/) that they released since the most recent time I set up a Windows installation. Feel free to pick another if you like, Chocolatey is just the one that was recommended to me so it's what I use.

You'll need to install a couple dependencies, run the following in your command line:
```
pip install requests
pip install mwparserfromhell
```
They actually provide a `requirements.txt` but I couldn't get it to work so not sure what was up with that, I didn't really try to debug at all.

#### Getting Git
[You may as well install Git](https://gitforwindows.org/). You'll need it eventually, trust me. (This is as opposed to installing via the `.tar.gz` or `.zip` that they provide.)

#### Cloning PWB
Make a folder called PWB and open git bash in it. The goal is to have your structure be `general wiki stuff/PWB/install here` so that you can have miscellaneous non-install stuff such as user-config backups etc outside of your install folder, but still categorized as general PWB stuff.

Now check that the instructions haven't changed from the [original mediawiki.org tutorial](https://www.mediawiki.org/wiki/Manual:Pywikibot/Gerrit#For_users) and run:
```
git clone --recursive --branch stable https://gerrit.wikimedia.org/r/pywikibot/core.git
```
Then `cd` into the `core` folder you just created.

### "Fixing" PWB code so that it doesn't break in WSL
We're following [this StackOverflow thread](https://stackoverflow.com/questions/32760041/pywikibot-how-to-handle-user-config-py-owned-by-someone-else/62986377) which I found when setting this up for the first time and then also commented a second change on.

Run the following git command before making these changes. This will put you in a separate branch so that you can later update master and merge the changes into your branch painlessly (in theory at least).
```
git checkout -b wsl-fixes
```

If you don't already have a preferred code editor, what editor should you use for these changes? There's a few options:
* [PyCharm community edition](https://www.jetbrains.com/pycharm/), the best Python IDE imo, but can be very confusing for new users, especially since the keyboard shortcuts suck out of the box (e.g. ctrl+N is not bound to new file...)
* [Notepad++](https://notepad-plus-plus.org/downloads/), the other side of the spectrum, the most hassle-free option, but also the most limited in features, and its default Python syntax highlighting is not the best
* [SublimeText](https://www.sublimetext.com/), this one makes sense [because of the Mediawiker plugin](https://river.me/blog/sublime-for-mediawiki/)
* [VSCode](https://code.visualstudio.com/), probably what most people will default recommend, but the one I'm the least-inclined to recommend out of all of these; I'm writing this blog post in VSCode because its markdown support is pretty great but I really don't like it that much

If you already use one of the above (or another text editor I didn't mention like Atom or something), pick that one; if you've never used one before then go with Notepad++ for the immediately-least-painful choice, SublimeText for the most-useful-for-wikis choice, PyCharm for the I-want-to-develop-in-Python choice, and VSCode for the screw-you-River-I-want-to-listen-to-everyone-else choice.

Okay, onto the actual changes.
#### Change 1
In `pywikibot/config2.py` change the following at line 987 (note that the line numbers may have changed since publication of this post) the following snippet:

```python
# Get the user files
if __no_user_config:
    if __no_user_config != '2':
        warning('Skipping loading of user-config.py.')
else:
    _filename = os.path.join(base_dir, 'user-config.py')
    if os.path.exists(_filename):
        _filestatus = os.stat(_filename)
        _filemode = _filestatus[0]
        _fileuid = _filestatus[4]
        if OSWIN32 or _fileuid in [os.getuid(), 0]:
            if OSWIN32 or _filemode & 0o02 == 0:
                with open(_filename, 'rb') as f:
                    exec(compile(f.read(), _filename, 'exec'), _exec_globals)
            else:
                warning("Skipped '%(fn)s': writeable by others."
                        % {'fn': _filename})
        else:
            warning("Skipped '%(fn)s': owned by someone else."
                    % {'fn': _filename})
```

should become:

```python
# Get the user files
if __no_user_config:
    if __no_user_config != '2':
        warning('Skipping loading of user-config.py.')
else:
    _filename = os.path.join(base_dir, 'user-config.py')
    if os.path.exists(_filename):
        _filestatus = os.stat(_filename)
        _filemode = _filestatus[0]
        _fileuid = _filestatus[4]
        with open(_filename, 'rb') as f:
            exec(compile(f.read(), _filename, 'exec'), _exec_globals)
```
#### Change 2
Then in `pywikibot/tools/__init__.py` in the following snippet:
```python
    warn_str = 'File {0} had {1:o} mode; converted to {2:o} mode.'
    if stat.S_ISREG(st_mode) and (st_mode - stat.S_IFREG != mode):
        # the following single line is the line I commented:
        # os.chmod(filename, mode)
        # re-read and check changes
        if os.stat(filename).st_mode != st_mode and not quiet:
            warn(warn_str.format(filename, st_mode - stat.S_IFREG, mode))
```
The line that is originally 1717 (again this may have changed since publication) needs to be commented.

### Configuring PWB
If you're configuring for Wikimedia projects, just follow the [official setup guide](https://www.mediawiki.org/wiki/Manual:Pywikibot/Installation#Configure_Pywikibot) at this point.

If not, then once again there are [instructions](https://www.mediawiki.org/wiki/Manual:Pywikibot/Use_on_third-party_wikis) but I'll provide my own.

1. Run `py pwb.py generate_family_file.py` (you should still be in a command prompt located inside your installation folder called `core` if you forked from gerrit earlier). You only need to do this for one of the wikis you want to be able to run PWB on for now.
2. Run `py pwb.py generate_user_files.py`. See [my post about bot passwords](https://river.me/blog/bot-passwords/) for advice on naming and granting rights to your bot password.
3. Set `put_throttle = 0` in your `user-config.py` if you want to retain any sanity whatsoever; this will stop your scripts from pausing between edits. If you are concerned about performance of the server your wiki is on, maybe you want like 1 second throttle but the default is insanely high.
4. [Semi-Optional] Open your `user-password.py` and manually add extra accounts as needed. It should look something like this:

        ("RiverIsABot", BotPassword("PWBBot", "REDACTED"))
        ("RheingoldRiver", BotPassword("PWBBot", "REDACTED"))
    I said semi-optional because you should **really** have both a human and a bot account if you're using automated software like PWB.
5. [Optional] Create additional family files either by running `generate_family_file.py` additional times as needed. You can also use the syntax `family_files['cod'] = 'https://cod-esports.gamepedia.com/api.php'` directly in your `user-config.py` but I found that was giving me an annoying error that [seems to not be addressed](https://phabricator.wikimedia.org/T130570) so I gave up on that convenience and just made family files for everything.
6. [Optional] Add the default usernames in your `user-config.py` for each of the family files you make, it will look something like this:

        usernames['lol']['en'] = 'RheingoldRiver'
        usernames['cod']['en'] = 'RheingoldRiver'
        usernames['smite']['en'] = 'RheingoldRiver'
        usernames['pala']['en'] = 'RheingoldRiver'
        usernames['gears']['en'] = 'RheingoldRiver'


## Windows Subsystem for Linux
### Installing WSL
Okay, this part is easy, just go to the [WSL Installation Guide for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10) and follow the instructions. Get Ubuntu. There's really no reason not to get Ubuntu. You can get either 18.04 LTS or 20.04 LTS. I'm using the latter.

As for deciding between WSL 1 and 2, I'm using WSL 1 by default because when I did the setup my version of Windows wasn't late enough to get WSL 2. I'm unconvinced WSL 2 is an improvement for this particular use case, since it's extremely convenient to keep all PWB files perma-open in a text editor like Sublime or Notepad++ and copy-paste lists of wiki pages from the browser to these files at will and then run commands on them; I don't really want to store these files in the Linux filesystem. So I would go with WSL 1.

(I'm pretty sure that if you did this entire setup within WSL, using WSL 2, then there wouldn't be any permissions errors and you wouldn't have to modify the code as we did earlier. If you try this, I'm interested to find out if this is in fact the case. But like I said, the mixed-filesystem approach is certainly a more *convenient* setup, even if possibly a "worse" one.)

You will have to restart your computer after installing WSL.

### Navigating to your installation
For me, I run:
```
cd /mnt/s/Documents/Wikis/PWB/Install/
```
My PWB folder is located on a drive called `S:` (S for Solid State, I don't have that many drives lol). You're more likely to look in `/mnt/c/Users/YourUsername/Documents` or something so you can `cd` to there and then browse with `ls` and find your way around. (Also, I renamed the folder `core` to `Install` after cloning PWB.)
## Fish shell
### Installing Fish
Simply follow the instructions from the [Fish shell maintainers team](https://launchpad.net/~fish-shell/+archive/ubuntu/release-3).

Now just type `fish` to enter the fish shell. It will say, "Welcome to fish, the friendly interactive shell." Yay, you should now have pretty colors and history autocompletion!
### Configuring Fish
To edit your aliases, run:
```
vi ~/.config/fish/config.fish
```
(One of the first aliases I would make is to alias this command itself. I call it `viconfig`.)

This command will have you use vim, to start editing press `i`, to stop editing press `esc`, after pressing escape, type the characters `:q!` to quit without saving or press `:wq` to save and quit. If you don't like vim or the idea of vim, [here's a guide to using emacs](https://www.digitalocean.com/community/tutorials/how-to-use-the-emacs-editor-in-linux). For help using Vim, I like [the Vim Tips wiki](https://vim.fandom.com/wiki/Vim_Tips_Wiki). (Disclaimer, I'm a contractor for Fandom, but I'm in no way connected to that wiki or benefit from you using it or clicking this link.)

To reload your config after editing it, just type:

```
source ~/.config/fish/config.fish
```
You can also alias this, though of course the **first** time you reload your config file, it won't yet be aliased. I call this `reloadconfig`. (Yes this is a rather long alias, but, history autocompletion!)

Here's my current set of aliases:

```
alias pwb "python3 pwb.py"
alias ptouch "pwb touch "
alias lolse "ptouch -cat:Pages_with_script_errors"
alias pwbdr "pwb redirect do -always"
alias pwbfam "pwb generate_family_file.py"
alias reloadconfig "source ~/.config/fish/config.fish"
alias viconfig "vi ~/.config/fish/config.fish"
alias pwbcd "cd /mnt/s/Documents/Wikis/PWB/Install/"
alias rmwc "python3 -m pip install -U git+git://github.com/RheingoldRiver/river_mwclient"
```

(Note that the alias `lolse` breaks convention because it's equivalent to my [browser alias](https://river.me/blog/url-bar-is-a-cli/) for loading the script error category on Leaguepedia, `pwbse` would make more sense but whatever, `lolse` was the first alias I made (actually it was originally `lolse.bat` when I used the regular command prompt) and I never changed it.)

You can see that the aliases cascade down; first I define `pwb` and then I continue to use it, then I define `ptouch` and continue to use it, etc. This is a good practice not just to be lazy but also in case your tooling changes in the future or something.

Also, your config can do things more advanced than just define aliases; you can also define functions that have different behavior depending on their inputs, though I won't explain how to do that here, read the Fish docs if you're interested in that.

Finally, when you run an alias, it's literally aliasing that into the shell; you can append further arguments if you want. For example, `lolse -family:cod` is a valid thing to run, as is `pwbdr -user:RiverIsABot -family:smite`, etc. So this is a very versatile thing to run!

## Conclusion
We went over how to install PWB, WSL, and the Fish shell. Of course you can stop early if you don't want this entire setup, so feel free to use this post as just a set of instructions on how to install PWB for third-party wikis, or use your own preferred shell if you don't like Fish.

Pywikibot is an amazing tool for its prebuilt scripts and CLI interface, but I would NOT encourage you to write your own scripts in its framework. Instead I'd check out the [mwclient](https://mwclient.readthedocs.io/en/latest/user/index.html) and [mwparserfromhell](https://mwparserfromhell.readthedocs.io/en/latest/) libraries and write your own code from scratch using these! One day I'll do a post about that, too....