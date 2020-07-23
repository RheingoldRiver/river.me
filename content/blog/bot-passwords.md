---
title: "Bot passwords & you (& your bot (& you))"
date: 2020-06-24T09:43:18Z
draft: true
tags:
    - mediawiki
summary: What's a bot password? What's the difference between bot accounts and bot passwords? Why do I need all this anyway?
---

A "[bot password](https://www.mediawiki.org/wiki/Manual:Bot_passwords)" is a [method of authentication](https://www.mediawiki.org/wiki/API:Login) that you can use to log into the [MediaWiki API](https://www.mediawiki.org/wiki/API:Main_page). If you want to read technical manuals, read those three links instead of this blog post; I'm going to take a non-technical approach here (!).

The confusion as to what a bot password, in my opinion, stems entirely from the fact that "bot password" is a really, really confusing name for it. The name exists for historical reasons; non-human accounts that perform edits are called "bot accounts," and so when humans make automated edits from their own accounts those are also referred to as "bot edits," and so how do you log in to perform automated edits? Why, with a "bot password," of course! Note that the logic is not bot account -> bot password; the logic is automated edit -> bot password. Your human (personal) account can have one or more bot passwords as well.

Note: I've previously contributed to the [Gamepedia help wiki page about bot passwords](https://help.gamepedia.com/Logging_in_to_third-party_tools), and some of the content here will overlap with content there (at least at the time of publication).

## So what actually is a bot password?
A bot password is a way to tell the MediaWiki API who you are without using your actual account's credentials. If you go to [Special:BotPasswords](https://www.mediawiki.org/wiki/Special:BotPasswords) while logged into any wiki on a version of MediaWiki at least as recent as 1.27, you will be given the opportunity to create, edit, or delete any bot passwords you might want.

A bot password is NOT:
* A password (you can't use it to log into the site) - but you should TREAT IT LIKE ONE because someone CAN use it to GAIN ACCESS TO YOUR ACCOUNT
* A bot (it is not software)
* An account (it's tied to your existing account)
* Unique - you can have MULTIPLE bot passwords per account

Later on we'll elaborate on the last point extensively.

## Components of a bot password
The terminology gets even more confusing here. Each "bot password" has both a "name" part and a "password" part. So there's the bot password name (aka the bot password) and the bot password...password. Here's a current screenshot of all my bot password names on Leaguepedia:

![Screenshot of all my bot password names on Leaguepedia: AWB, Nami, PWBBot, Python, PythonRemote, and ST3](/images/bot-passwords/password-names.png)

As you can see, there is no sensitive information here. The actually secret part, the token part, is available when you create the password on a visible-only-once screen that you can copy-paste it from. After you see this screen one time, you're unable to get it back; if you lose the password part, you have to recreate it.

## Bot accounts & human accounts

In the introduction, I mentioned that the name "bot passwords" is derived from the practice of using bot accounts to perform automated edits - but why even bother doing this? It's all about avoiding polluting recent changes.

Let's take a look at the recent changes filters on Wikipedia:
![Screenshot of recent changes filters on Wikipedia](/images/bot-passwords/recent-changes.png)
The first filter in particular is of interest: "Human (not bot)." In other words, we're filtering out edits that are tagged as "bot" edits. Bot edits tend to be things like mass template or parameter renames, automated typo fixes, whitespace adjustments, and the like - high-volume, low-impact, low-interest edits that someone who wants to stay current with the wiki's "actual" changes is likely to consider to be "irrelevant spam."

If we want to give the "bot" flag to an edit, we need the "bot" [user right](https://www.mediawiki.org/wiki/Special:ListGroupRights). The bot user right is given exclusively by the bot user group.

To reiterate, there are three things:
* The `bot` flag can be given to an edit
* by a user with the `bot` user **right**
* which is typically granted to the `bot` user **group**, but this is configurable in LocalSettings; see [Special:ListGroupRights](https://www.mediawiki.org/wiki/Manual:Bots).

Sometimes, you may want your automated edits NOT to be hidden from Recent Changes by default. Since most automated tools will ALWAYS apply the `bot` flag when possible (i.e. when the account making the edit has the `bot` user right), this means that you must edit from an account WITHOUT this right when you want the edit to show up by default. As a result, many editors will have two accounts: one that is their "own" account, a human account, and one that is their "bot's" account, which will be part of the `bot` user group (and therefore have the `bot` user right).

Both the human account and the bot account will have their own set of bot passwords. I hope you are following along!