---
title: "Bot passwords & you (& your bot (& you))"
date: 2020-07-28T09:43:18Z
draft: false
tags:
    - mediawiki
    - best_practices
summary: What's a bot password? What's the difference between bot accounts and bot passwords? Why do I need all this anyway?
---

A "[bot password](https://www.mediawiki.org/wiki/Manual:Bot_passwords)" is a [method of authentication](https://www.mediawiki.org/wiki/API:Login) that you can use to log into the [MediaWiki API](https://www.mediawiki.org/wiki/API:Main_page). If you want to read technical manuals, read those three links instead of this blog post; I'm going to take a non-technical approach here (!).

If this is a non-technical article, but also not a brief tutorial, then who is it for? This is intended for anyone who has to work with bot passwords frequently and occasionally feels like they are just reciting the components of working with one, rather than really understanding each part, or falters when trying to explain it to someone else.

This article is rather indepth for what should maybe be a very brief and simple concept, but I think the terms are so poorly named in MediaWiki that a full explanation is warranted to prevent any lingering confusion that could lead to mistaken beliefs. Also, a deep understanding is the best way to remove a need for easily-forgotten memorization, so I think the detail here is appropriate.

Note: I've previously contributed to the [Gamepedia help wiki page about bot passwords](https://help.gamepedia.com/Logging_in_to_third-party_tools), and some of the content here will overlap with content there (at least at the time of publication).

## So what actually is a bot password?
First of all, the name "bot password" exists for historical reasons and not because the name is descriptive; non-human accounts that perform edits are called "bot accounts," and so when humans make automated edits from their own accounts those are also referred to as "bot edits." How do you log in to perform automated edits? Why, with a "bot password," of course! Note that the logic is not bot account -> bot password; the logic is automated edit -> bot password. Your human (personal) account can have one or more bot passwords as well.

Thus, a bot password is a way to tell the MediaWiki API who you are without using your actual account's credentials. If you go to [Special:BotPasswords](https://www.mediawiki.org/wiki/Special:BotPasswords) while logged into any wiki on a version of MediaWiki at least as recent as 1.27, you will be given the opportunity to create, edit, or delete any bot passwords you might want.

A bot password is NOT:
* A password (you can't use it to log into the site) - but you should TREAT IT LIKE ONE because someone CAN use it to GAIN ACCESS TO YOUR ACCOUNT
* A bot (it's not software, it's just a string)
* An account (it's tied to your existing account)
* Unique - you can have MULTIPLE bot passwords per account

Later on we'll elaborate on the last point extensively.

## Components of a bot password
The terminology gets even more confusing here. Each "bot password" has both a "name" part and a "password" part. So there's the bot password name (aka the bot password) and the bot password...password. Here's a current screenshot of all my bot password names on Leaguepedia:

![Screenshot of all my bot password names on Leaguepedia: AWB, Nami, PWBBot, Python, PythonRemote, and ST3](/images/bot-passwords/password-names.png)

As you can see, there is no sensitive information here. The actually secret part, the token part, is available when you create the password on a visible-only-once screen that you can copy-paste it from. After you see this screen one time, you're unable to get it back; if you lose the password part, you have to recreate it.

## Bot accounts & human accounts

In the introduction, I mentioned that the name "bot passwords" is derived from the practice of using bot accounts to perform automated edits - but why even bother doing this? It's all about avoiding spamming recent changes.

Let's take a look at the recent changes filters on Wikipedia:
![Screenshot of recent changes filters on Wikipedia](/images/bot-passwords/recent-changes.png)
The first filter in particular is of interest: "Human (not bot)." In other words, we're filtering out edits that are tagged as "bot" edits. Bot edits tend to be things like mass template or parameter renames, automated typo fixes, whitespace adjustments, and the like - high-volume, low-impact, low-interest edits that someone who wants to stay current with the wiki's "actual" changes is likely to consider to be "irrelevant spam."

If we want to give the "bot" flag to an edit and mark it as low-interest, we need the "bot" [user right](https://www.mediawiki.org/wiki/Special:ListGroupRights). By default, the bot user right is given exclusively by the bot user group.

To reiterate, there are three different meanings of the word "bot" here:
* The `bot` flag can be given to an **edit**
* by a user with the `bot` user **right**
* which is typically granted to the `bot` user **group**, but this is configurable in LocalSettings; see [Special:ListGroupRights](https://www.mediawiki.org/wiki/Manual:Bots).

Sometimes, you may want your automated edits NOT to be hidden from Recent Changes by default. Since most automated tools will ALWAYS apply the `bot` flag when possible (i.e. when the account making the edit has the `bot` user right), this means that you must edit from an account WITHOUT this right when you want the edit to be labeled as human. As a result, many editors will have two accounts: one that is their "own" account, a human account, and one that is their "bot's" account, which will be part of the `bot` user group (and therefore have the `bot` user right).

Both the human account and the bot account will have their own set of bot passwords. I hope you are following along!

I'm going to restate the above in a different order:

Most software that performs automated edits, like AutoWikiBrowser, will always attempt to apply the `bot` flag to every edit, relying on the fact that this attempt will fail when the account editing lacks the `bot` user right/group (let's assume these are the same, though of course this could be configured differently in LocalSettings). So to work within this ecosystem, editors will often have two separate accounts, one which is themselves and lacks the `bot` user group, and one which is their bot, and does have the `bot` user group.

Pretty much all standard software (at least out of what I use) - AutoWikiBrowser, PyWikiBot, the `mwclient` Python library (by default) - adheres to this convention of always attempting to send the `bot` flag, and so it's often seen as an equivalence that `bot` flag == `bot` user right == `bot` user group. However, these things are all different!

So to summarize, one person can have multiple accounts, one with bot rights and one without; each account can have multiple bot passwords; and each bot password will have a password name and a password secret.

## Using a bot password
![Screenshot of custom rights bundles at Special:BotPasswords](/images/bot-passwords/awb-login.png)
Usually, an application like AWB will want you to give it a user name in the format `AccountName@PasswordName` and then a password which is just the secret of the bot password. Shown are my saved logins for AutoWikiBrowser - as you can see, I remember the logins for both myself and my bot.

One could imagine a different paradigm in which my logins were `RheingoldRiver@AwbAsMe` and `RheingoldRiver@AwbAsBot`, and `FlagAsBot` were a right governed individually by each bot password - or even just a single login, and a checkbox in the AWB software of "show edits in recent changes" which toggles the addition of the `bot` flag to all my edits.

Even better would be an option "hide from recent changes after []" and then you could fill this in with a `0` to add the `bot` flag to all edits starting immediately, or, say, `20` to add the `bot` flag to edits starting with the 21st, etc.

While this paradigm isn't unlikely to exist ever, and this is just an exercise in frustration with limitations of the current system, understanding how it COULD work can help you understand the way it works now.

## Bot password best practices
Now that we (hopefully) understand bot passwords and rights and groups and flags, let's talk about some best practices with bot passwords!

### Unique usage per password
Each password should be used for exactly one application on one machine. You should have a password for AWB on your home machine, a password for PWB on your home machine, one for PWB on your work machine, etc. This has security benefits - if one machine is compromised, it doesn't compromise your other passwords. It also allows you to segregate rights per application more easily.

### Naming your passwords
Name your passwords consistently. For example, you might choose AppNameMachineName. I've had issues with passwords with underscores in them, so avoid underscores. I think it's fine to give your bot account a whimsical name, but for your bot passwords, keep them application-specific and focused - even if you're only making one single password right now, they have a tendency to multiply. Keep them as disciplined as you would variable names in your code!

### Assigning rights
Always assign the minimal amount of rights possible. In particular, be careful when allowing high-volume editing; in this case, you should avoid allowing the permission to edit sitewide CSS and JS. If you anticipate the need to edit CSS and JS via the API, use a separate bot password for it.

## Custom rights
This has been particularly relevant to me when using [SublimeText](https://river.me/blog/sublime-for-mediawiki/) to edit widgets - the `editwidgets` right isn't available via any of the default permissions you can select from `Special:BotPasswords`.
![Screenshot of custom rights bundles at Special:BotPasswords](/images/bot-passwords/custom-rights-bundle.png)
Fortunately, for this situation or similar, there is a very easy workaround! Simply create a custom user group in LocalSettings and assign the rights you need to this group. Make yourself (and anyone else who needs the rights for their bot passwords) part of this user group. Then, in `Special:BotPasswords` there will be the option to assign rights from one of your "rights bundles" for your custom user groups. Check this box, and you will be good to go!