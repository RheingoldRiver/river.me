---
title: "New Roster Changes"
date: 2020-01-01T18:03:16Z
draft: false
tags:
    - leaguepedia
    - mediawiki
    - cargo
summary: An overview of the process I used to create our single-source-of-truth roster change database tables on Leaguepedia. This post originally was created as a [user page](https://lol.gamepedia.com/User:RheingoldRiver/Blog/New_Roster_Changes) on Leaguepedia.
description: An overview of the process I used to create our single-source-of-truth roster change database tables on Leaguepedia.
---
Adapted from a [user page](https://lol.gamepedia.com/User:RheingoldRiver/Blog/New_Roster_Changes) blog post on Leaguepedia made on January 1, 2020. Reformatted for Markdown from MediaWiki on February 18, 2020.

TL;DR: I wrote some code. But you already knew that.
## Introduction
So this story starts about two years ago. I made a Cargo table for our news that stored basically 3 pieces of information - the news's subject (either a tournament or a team), the region the news was tagged to, and the text of the news. Technically there was another column called "SubjectType" that informed if the subject should be styled like a tournament or a team, but basically it was just the three data points. I was really excited about this table! It let me make region-specific news pages, which I knew could one day be leveraged to create region-specific versions of the front page (which are *still* on my todo list............) But, that's about all it let me do. There was no tagging to players or teams or tournaments, so I couldn't do queries that way to show news. There also was no semantic meaning to anything - it was just a text blob that the wiki could never parse information from. Basically, this wasn't even a proof of concept - it was more like proof of me wanting this.

Fast forward to last year. I spent all of December working on a HUGE SCARY AMBITIOUS PROJECT which I documented [here](https://lol.gamepedia.com/User:RheingoldRiver/Blog/Tournament_Cargo). By the time I finished, I knew that next year (aka this year) I was going to do the same thing - a huge ridiculous project with extremely large scope - in the offseason. And I knew it was going to be roster changes. That was basically all I knew.

Throughout the year, I spent some time here and there sketching ideas and thinking about design and stuff. What exactly was going to be the scope of what got replaced? There are a LOT of places we use roster change data. Here's a list:
* Front page news
* Roster change portal (sometimes)
* Roster change portal - current rosters tab
* Team pages - timeline
* Team pages - current players
* Team pages - former players
* Team pages - navboxes (though these have been automated from current rosters for over two years now, that was one of the first things I did at Gamepedia)
* Player pages - current team
* Player pages - team history
* Region pages (lol)

And there's some more places it might be neat to show rosters that we don't currently (though some pages have been converted to the new format as of me writing this list!):
* Player pages - timeline
* Tournament page roster section should show changes that happened during the event
* On team pages, where you see players' "next team" in roster section, you should be able to click to see a pop-up of their entire team history
* Same thing could maybe be done in timelines with an `::after` for players
* These [infographs](https://lol.gamepedia.com/Team_SoloMid/Team_Infograph) we used to have? I wonder if I could recreate these with CSS.......hm............It *might* be possible. So far I haven't done any mock-ups even, so I'm super unsure honestly. The hardest part would be role swaps I think. But I might be able actually to do this. (Obviously I won't design the CSS, Ema would, but like, maybe I can code them lol)

So as you can see the scope is ENORMOUS (and still unknown).

## Interlude - External Content
In May and June roughly I think, I somewhat reluctantly did a large project which was databasifying all of our external content. This is stuff like player interviews, articles about teams, etc. All of it used to be entered by hand onto player, team, and tournament pages (sometimes also champion pages). We wanted to "single source of truth" this (this is a common theme in everything I've done the past two years) - aka have SINGLE place on the wiki where data is entered, and every additional display will query that first place. There's a lot of reasons SSOT is nice, and you can read a lot about it if you search, but there's only one reason I particularly care about: it takes twice as long to update 2 places as it does to update 1. I want people not wasting time updating in multiple locations, I'd rather write code to multiply the singular effort.

How is a list of articles and interviews about/with someone similar AT ALL to roster changes? Well, a very important reason: the primary well-defined identifier of each is the date at which it happened.

Huh?

Well-defined means that for each input, there is only one output. Never two or more. A roster change involved 2 players? Player isn't well-defined. A piece of content is about 2 teams? Team isn't well-defined. Two authors collaborated? Author isn't well-defined. Two sources need to be listed for this change because each contains partial information? Source isn't well defined. But when did the roster change happen? When was the content released? That one is never changing. There's no debate. (Okay well there's GCD announce followed by team announce, and there's time zone issues, and and and....but you get the idea.)

So despite my reluctance to do this project at this time, it turned out to be a FANTASTIC learning experience and provided me with a solid interface framework for dealing with time-associated data points. I designed a two-level navigation in which the first level is a list of years, and the second level is a list of Sundays in a single year. Navigate within a year by clicking Sundays. Each page has a full week of data on it. Pages are named after the date of the Sunday, but the other 6 dates of the week redirect to the Sunday, so you can directly type a URL if you know the date. Later on, I added a final layer of navigation, which takes you between the (so far) three types of date-centric data (external content, roster changes, and roster rumors).

Within each date, users can simply add a line of information, and the code propagates the date to the items placed underneath that date. There's also an internal index, so that queries preserve the ordering within a date. I wrote a Python script to generate a full year (or even a range of years) of dates along with the navbox code and the redirects. This system will probably be in place for at least the next decade. I love it unconditionally.

You can see it [here](https://lol.gamepedia.com/Data:News/2019-05-26).

## Model 1: Data Entry
We are now in August 2019.

The first data model I had to design was how people would do data entry. It was of vital importance that this be my first data model, and the reason is, I needed people to start data entry before I could finalize much of anything - remember that I had very little idea what the scope of this project was still. I obviously knew that changes were going to be associated to date and nothing else. But how would users input?
### How to Group Changes
There were a few options (some of them obviously sucked but I'll list a bunch here anyway):
* Input all the changes a player made on a day, grouped together
* Input all the changes a team made on a day, grouped together
* Have one template for departures and another template for joins
* Have one line per team per player

Actually, all of these sucked. Remember the goal is to minimize user data entry. It was pretty quickly obvious to me that the correct answer was to group by *team and source*. That is, if a team made two announcements in one day, there would be two entries for that team that day. If a team made one announcement where they announced 10 players, there would be 10 lines. And so on.

Later on we amended this a bit - so there's actually support for having multiple sources listed together, the catch is that in this case all of the sources will be associated to all of the players that moved in that group of sources. So when it's included on one player's page, there may be 5 links, only one of which is related to him. But that's a reasonable compromise in some places where the players made announcements instead of the team making a single announcement.

### How to Enter Changes
So I'm grouping by source, but that still is only half the question. How exactly are people supposed to express the change? We obviously can't just write in English, "Spica joins. Johnsun moves to substitute." We could have a list of players and their changes, for example `{{PlayerChange|Spica|join}}`. But what if the player is role swapping? Moving to substitute? Moving from substitute to trainee? This invites a lot of possible change verbs. So I can't do it just by verbs. What about before/after? You could imagine |rolebefore=top |subbefore=yes |traineeafter=yes but this seems to invite a lot of long parameter names and seems pretty annoying to remember.

Instead I abandoned the idea to group by player within team. Instead I furthered the analogy of grouping by *time*, and created `|before=<list of players> |after=<list of players>`. The idea is, in the `|before=`, you write all of the players in how their statuses were before the transaction in question happened (n.b. this could be null). And then in `|after=`, you write the same thing but for after. So a join would look like `|before= |after=player`, etc. Actually the player isn't just a player, it's a player plus an optional status flag plus role plus optional role modifier (sub/trainee) plus.......well, you can see all the params here:
```
{{{player|}}};;;{{{role|}}};;;{{{status|}}};;;{{{loaned_from|}}};;;{{{loaned_to|}}};;;{{{move_type|}}};;;{{{custom|}}};;;{{{contract_until|}}};;;{{{assistance|}}};;;{{{event|}}};;;{{{replacing|}}};;;{{{reason|}}};;;{{{phase|}}};;;{{{sub|}}};;;{{{trainee|}}};;;{{{rejoin|}}};;;{{{order|}}};;;{{{sentence_group|}}};;;{{{leave_date|}}};;;{{{nolink|}}};;;{{{remain_for|}}};;;{{{remain_for_link|}}};;;{{{already_joined|}}};;;{{#or:{{{team_priority|}}}|1}}:::
```

(I'm sure more params will be added in time, too.)

I made this template as generic as possible so that it can be used in multiple places, and it's currently also used in the portal current rosters before list (more on this later); likely I'll use it in more places too. Of course, I didn't come up with this list immediately, and it evolved a lot. Originally there was a combined `|status=` that included sub, trainee, inactive, etc. But then of course you can be a sub AND inactive, a sub AND set_to_leave, etc. So these couldn't be together. Now, sub and trainee are special because those both modify role. And currently those are the ONLY two modifiers of role available. So I did the natural thing and added `|sub=` `|trainee=`. WRONG!!!!!!!!!!! This was stupid and terrible because imagine one day there's, like, reserve players. Then you could have Trainee OR Substitute OR Reserve, and all 3 of these could modify! And I was HARD-CODING THIS SHIT INTO MY DB TABLES??????????? God no. So needless to say, I redid the entire thing again, and I think I went through 3 iterations total of sub/trainee, and my current setup is `RoleModifier`. (Actually, the template still accepts `|sub=` `|trainee= ` but it's stored in the db as RoleModifier and that's what my backend knows - I debated changing it in the user-facing template too, but people are used to putting `|sub=` `|trainee= ` for so long that I'll stick with this on user-facing end until a new option is needed. As long as my code immediately casts this to RoleModifier it's ok. So the user-facing went through two iterations, and the database through three (for sub/trainne specifically).)

Some of these parameters are yes/no flags, some want data...but the vast majority of them are only needed in a few special circumstances, and I wrote pretty detailed docs that are actually built into my code (!!!) so they will stay updated:

```lua
        join = { sentence = '$PLAYERS$ $JOIN$.', desc = "Standard players joining", howto = "Automatic", auto = true, respect_joining_key = true, class = "join" },
        join_as_temp_sub = { sentence = '$PLAYERS$ $JOIN$ as temporary $SUB$ for $PHASE$$EVENT$$REPLACING$$BECAUSE_COMMA$$REASON$.', desc = "A player joins as a temp sub", howto = "|status=temp_sub", auto = true, respect_joining_key = true, class = "join" },
        leave = { sentence = '$PLAYERS$ $LEAVE$.', desc = "Standard players leaving", howto = "Automatic", auto = true },
        loaned_from = { sentence = '$PLAYERS$ $ARE$ loaned from $LOANED_FROM$.', desc = "", howto = "Set |loaned_from= as the team they're officially on and |status=on_loan", auto = true, class = "join" },
        loaned_to = { sentence = '$PLAYERS$ $ARE$ loaned to $LOANED_TO$.', desc = "Players loaned out", howto = "Set |loaned_to= as the team they're temporarily going to and |status=loaned_out", auto = true },
```

You can see the full docs (including some manually-written stuff outside of the scope of the auto-generated ones) here: [Template:RCPlayer](https://lol.gamepedia.com/Template:RCPlayer)

## Model 2: Database
### Storing Transactions
Despite how I said there were a lot of potential paradigms for how users enter data, actually I decided on the general idea of the final method relatively quickly. The same decision for the database was...not as easy to decide on. Actually I reworked this extensively partway through. Here's some things I knew from the start:
* I have to write an English sentence pre-storing and have that attached to the relational data.

Hahahaha yeah that's it. That's the only thing I knew before I started. But I did figure out some other things pretty early on:
* I need a one-to-many relationship between news and individual roster changes.
* There has to be some uniformity to how leaves and joins are handled so that I can do one query and get both, and know what I got.
* I'm going to need *multiple* tables that can join to parent table News to provide a well ordering of events that happened: for example, retirements, players renaming, etc.
* The **display** date and the **order by** date need to be decoupled - one might be known with *significantly* more accuracy than the other (e.g. December 1, 2019 vs 2018 (approx) could both be entries in a team's timeline).

My first design used the following method:
* Store before status and after status in each row
* Some rows will have both before and after (e.g. role change)
* Others will have one set or the other be null
* Never actually bother to state whether this is a leave or a join, we can infer this from the set of null parameters

I wrote a bunch of code using this method. But then I got to a problem, specifically in player team history where a player had a role swap on a team. This meant that the row should be used as a "leave" in the one row, and then as a "join" in the row after. But because I was assigning all parameters of a query result row to a player information output row, I had the side effect that the "leave" was present in both roles. Not good. I thought about putting some conditional logic to fix that, but my goal has been to avoid as much conditional logic as humanly possible, and just have the database set up to make everything super easy. So I decided to change how it was stored a bit: Instead of each row being the player's entire change (leave (if applicable) to join (if applicable)) it would just be a leave OR a join. If a player was role swapping, in the database I'd store this as a leave of the old role followed by a join of the new row. My code doesn't need to know the difference! The only time this might be an issue is if I'm showing players' entire tenures on a team, but:
* On player pages we break this up by role anyway
* On team pages I can just completely ignore all role swap preloads (more on this later)

Unfortunately..............this was not fantastic, and if you've worked with databases you'll notice a huge red flag: this new system meant that in every row, there was a GUARANTEE that half of my columns would be null, and, *ew*. But we're not there yet.

Before I fixed this travesty, I had another more pressing issue. I've been using the words "join" and "leave" but in reality at this point in time my database had the words "Start" and "End." YIKES. Why yikes? Let me share this comment that was in my code....
```lua
    -- remember Start is the team at the Start of the change and End is the team at the end
    -- so the idea of start and end is equivalent to the portal concept of this
    -- and is the opposite of the teamhist use of it
```

Yeahhhhhhhhh so Start and End were great when i was making the portal, where changes go from where you started to where you ended up, but...player team history tracks the OPPOSITE, the time you spend ACTUALLY ON the team, or in this case, from End to Start (joining a team means that team is saved as the End, and leaving a team means that team is saved as the Start). So this was obviously awful and needed to go, and I spent a day renaming all of my parameters and it was great.

THEN after that I got irritated at the shitty organization of having all of this stuff split out into a leave and a join, and I changed it to save one copy of each thing, plus a new field called Direction which could be either Join or Leave. To recreate all of the fields that my code expected, I concatenated that direction before each key value. Would it have been better to rewrite everything at this point to have a strucutre of `{ Join = {}, Leave = {} }`? Yes, but it was certainly not worth it to spend the time - I have way too much code to write total and at this point the kinda-shitty data structures are encapsulated, so goodbye see you again never.

### Storing Semantic Data
In addition to the transactions, I have to somehow ascribe **meaing** to all this shit. Thankfully, I have a list of like 30 preloads that get set based on what the user enters! (See [Template:RCPlayer](https://lol.gamepedia.com/Template:RCPlayer) So instead of doing magic query-side, I just need to make it so that `set_to_leave_already_joined` (the player is set to leave a team but already joined the new team, and is just remaining with the old team for one last event) is handled materially differently from `set_to_leave`. But recall I don't want to do ANY kind of conditional logic, therein lies disaster explosion. So, instead I did my favorite thing ever and created a settings data structure:

```lua
local PRELOADS_TO_IGNORE = {
    'opportunities', 'set_to_leave', 'remain', 'extended', 'gcd_remain', 'gcd_extended', 'loan_end_and_join',
    join = { 'set_to_leave_already_joined', 'to_academy_also_stay', 'to_main_also_stay' },
    leave = {},
}
```

The preloads in the top-level list are all ignored unconditionally. The tables `join` and `leave` are only ignored when they are sending data of that phase. For example, `set_to_leave_already_joined` has two lines associated with it: The first is leaving the team as a regular player, and the second is "rejoining" the team as a `set_to_leave` player. But, actually, in roster portals specifically we want to ignore the `join` part of this row completely: We want to show them as completing the transition from old team to new cleanly, because that's what's confirmed to happen for this offseason period! In a player team history, we'd not want that at all - instead we don't care in the slightest that a `set_to_leave_already_joined` occurred; we won't care until the player actually leaves the old team. So, for Player Team History we have the following settings instead:
```lua
local PRELOADS_TO_IGNORE = {
    'extended', 'gcd_extended', 'remain', 'expire_notleave', 'set_to_leave', 'opportunities', 'set_to_leave_already_joined',
    leave = {},
    join = {}
}
```
Note the difference of position of `set_to_leave_already_joined`. Hooray, semantic data!

I'll confess, I didn't actually believe this would be sufficient; I thought I *would* end up requiring some conditional clauses. And it's still possible I will end up there, but for now actually these settings tables are completely sufficient for ensuring each query has precisely the data I want it to.

## Model 3: Queries
This name is kinda misleading, there's currently four totally distinct data models that are generated by querying this stuff.
### Query 1: Grouped Roster Changes
[Example Portal](https://lol.gamepedia.com/Roster_Swaps/2020_Preseason/North_America); [Example Player](https://lol.gamepedia.com/Doublelift)

Player team history and roster change portals, share a lot of elements in common: You get a full list of applicable changes, and then go through them combining rows that ought to be grouped together and throwing out rows of duplicate data (for example, you don't want to show anything for a GCD confirmation of a previously-announced-by-the-team departure). So I made an abstract class [Module:GroupedRosterChangesAbstract](https://lol.gamepedia.com/Module:GroupedRosterChangesAbstract) and subclassed to create the others. Actually I need to further subclass [Module:PlayerTeamHistory](https://lol.gamepedia.com/Module:PlayerTeamHistory) so I can have a version that does a slightly different display for a pop-up display, but that's just a small change.

I went over that pretty quickly, but writing this was probably the hardest code to write (not counting architecting as writing code). It has a lot and a lot of moving parts. It's the kind of thing where if you told me I had a crucial flaw in a case that occurs for .1% of players on the wiki (so 7 total) I would totally believe you, and I'd cry, because I don't want to change it. But actually it could have been a lot worse than it is, function names like `updatePlayerChangesAndIndexNewEntry` and `updatePlayerChangesForExistingLine` actually do help....
### Query 2: News
[Frontpage](https://lol.gamepedia.com); [Example Team](https://lol.gamepedia.com/Team_Liquid#Timeline)

This query groups things together strictly chronologically, and oh man that is so much easier than "well...by the line that it makes sense to associate it with...." I also used an oo approach here, with [Module:NewsQueryAbstract](https://lol.gamepedia.com/Module:NewsQueryAbstract) and versions for players/teams. The by-date grouping is actually pretty different when you need the extra by-year grouping for teams, and players need to bold the link to their own page, etc. Not really anything particularly interesting here. These queries also ONLY query the news table, instead of joining to other tables as well (English sentences are stored in News because they're common to all news-related tables).

### Query 3: Portal Current Rosters
[Example Page](https://lol.gamepedia.com/Roster_Swaps/2020_Preseason/North_America/Current_Rosters)

These pages were pretty unique: I needed the *user* to supply the before-data, use a query to gather the after-data, and then write code to interpolate the two. This was my second use of the [Template:RCPlayer](https://lol.gamepedia.com/Template:RCPlayer) template mentioned above. Once again I needed to associate two statuses together, but this one was significantly easier - I don't actually care about any diff between queried changes, only between the user-entered status and the most recent queried status. So instead of doing a bunch of index manipulation and remembering and passing and organizing, I just had to overwrite the "after" status every time I got updated news from the query. Whew so easy!

Later on I added a join to [Table - Players](https://lol.gamepedia.com/Special:CargoTables/Players) so that I could fetch residency. It's not a *fancy* thing and definitely some residencies will end up outdated in the future when people look back at these pages, but to do any differently would've been an actually insane amount of work, and this is still pretty useful. I left a note on the page explaining the potential discrepency.
### Query 4: Current Team
The final query is to determine player's current team (or teams). This is an even easier version of the portal current rosters query, so I just kinda did it and it was like 100 lines of code or something really short.

## Future
I wanted to write this now because of the start of the new year, and also honestly because I'm already starting to forget some of the things I did. But I still have a bunch of things to do!
### Team Rosters
I still haven't decided exactly how I want to show current and former team members. Currently I have a calculated table called [Table - Tenures](https://lol.gamepedia.com/Special:CargoTables/Tenures) that stores on player pages. Now, I definitely DO want this table, because it allows for a new type of query that we could never do in the past, which is "who has been on a team for the longest period of time?" We can ask that per team, per region, about all imports in a region, etc. And in the past we could NEVER answer it - but once we've converted all data, we will be able to. But, do I actually want to be querying this table on team pages? I'm not sure. It's certainly the *easier* way to determine past team members, but it adds a dependency I don't quite like - player pages need to be blank edited *before* team pages are purged in this case. And team pages *already* store current rosters, which shouldn't be moved elsewhere, and navboxes need to query those. But the alternative is to do two queries on team pages (first get all players that ever played on this team and then query to get their entire transactional history including renames, residency changes, everything), and it'll involve a lot more code. But that might be worth it just to avoid the dependency cycle of team needing Tenures from player but player needing CurrentMembers from team for the navbox.

### Player Team History Pop-Up
I mentioned this before, but I want to have a click-to-show in team former member lists that shows the player's full team history.

### Tournament News
We will add tournament-specific news on tournament pages. Since I'm not planning on making everyone tag a `|tournament=` in news, this'll be done by querying all teams associated to this event and joining that to news in the applicable date range.

## Bonus
### Roster Rumors
In the first week of November I took a temporary detour to build [Table - RosterRumors](https://lol.gamepedia.com/Special:CargoTables/RosterRumors). It was a quick throwaway thing that I just wanted to do so that we weren't entering data manually by hand for rumors anymore; it also significantly improved discovery for all of the lines that needed to be updated when a player was confirmed somewhere as well as eliminating the worry of multiple-region rumors going half updated. It also added a nice query form for people to filter by rumor author, player, etc. I wanted to write some code that tied this table to the roster change table, so that for example you could click a pop-up to see all roster rumors pertaining to any player in a portal who has not yet joined a new team after leaving their old team. I didn't get to that, but probably next offseason (so midseason 2020) I will.
### Random Pitfalls
Here's an assortment of things I struggled with along the way:
#### Contracts
How to store a player's contract's end date???????????? As another table that can join to News, with the display date being the date the contract was added to the GCD if known, otherwise empty and a sort date of, pick any day may as well be today. This seems...really obvious in retrospect, but because this information was shown in team roster only before, I couldn't really decide what to do with it.
#### Roles
I'm still using a field whose documentation is `Temp legacy support`. That should tell you everything you need to know.

But actually, this is pretty annoying, I need to store something that's unformatted and then re-format it every time I query, but someone could have multiple roles and they might be ingame-only or they could be staff or a combination of both and I also need to do SOME processing prior to store so that I'm storing consistently....Eventually I'll fix this and delete that field. Eventually.
#### Cargo Store Namespace
This is just a facepalm story really. After I wrote like 5 modules of code I realized that I wasn't specifying data namespace for any of the cargo_stores. This is really bad on my wiki because I have a gadget that automatically saves a user page every time you get an edit conflict so that there's no chance of getting confused and accidentally navigating away and losing everything - the backup is the automatically-created Edit Conflict sandbox page, which a cron job will then delete at the end of each day. A side effect of this backup is that there may be several user-page copies of any given data page. So it's *imperative* that I'm checking namespace before I store. I kinda didn't want to rewrite all of my stores to check for namespace so I kinda....put executable code in one of the utility modules I was using for news stuff.................

Let's just say that was a *very* frustrated half hour I spent debugging later on, and I fixed it. The sad part is I felt so shit about this entire thing that I was going to fix it anyway, and I STILL got fucked because of it. Yeah let's never do that again.

## Conclusion & Thanks
I spent pretty much the entirety of January 1st writing this. My wrists kinda hurt now....But I like keeping logs of cool projects I do and I hope you enjoyed reading it!

Special thanks go to Thrust for being with me on this from day 1 when I had nothing more than 50% of 1 module written and a promise that "this will be really cool and worth it I swear." Also to Figo, Andrew, Snackbar, Epamynondas, and James for going through the data entry part of this, and Jibril for doing all of the roster rumor and portal stuff. Thanks to Jesto for forcing me to do the content thing in May that I really REALLY did not want to do, it was totally worth the time & I don't think New Rosterchanges would've been nearly as successful as it was if I didn't already have that data entry structure built. As usual thanks to Ema for doing css because I can't design shit. To Yaron for making Cargo god I love Cargo it's the best, I can't imagine a wiki without Cargo (yes I did have one for like 4 years shhhh it was less nice). And to everyone who edits the wiki and/or listens to me rant about how much I hate everything at 3am because my code doesn't work until 5 minutes later when it does and suddenly I love the wiki again.

Happy 2020!

## Modules Referenced
These are just DPL queries of categories restricted to module namespace, so it's possible some things will be missing. I'm not the best at remembering to categorize.
**Query Modules:**
* [Category:News Templates](https://lol.gamepedia.com/Category:News_Templates)

**Storage Modules:**
* [Category:News Data Templates](https://lol.gamepedia.com/Category:News_Data_Templates)