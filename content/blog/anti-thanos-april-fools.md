---
title: "The Anti-Thanos Snap That Wasn't"
date: 2020-03-31T01:08:42Z
draft: false
tags: 
    - leaguepedia
summary: The story of the April Fools Day joke for 2020 that I didn't do, and the "Snap" that we may or may not still do.
description: The story of the April Fools Day joke for 2020 that I didn't do, and the "Snap" that we may or may not still do.
---

In about August 2019 I decided that our April Fools Day joke for 2020 was going to be to put a banner saying, "Great news! We have abandoned all notability guidelines on Leaguepedia!" The joke was....it was going to be true.

Internally, we called this "The Snap," which evolved from a comment that "this is like pulling a reverse Thanos."

## Background: The Fortnite esports wiki

On the *Fortnite* esports wiki, we track players' power rankings. Power rankings points are determined by wiki staff, and players receive points for placing in events. The main draw of the wiki is the set of power rankings pages, far outweighing the interest in individual player pages. As a result, most of the player pages are relatively low-content, and they can easily be automatically generated with approximately three data points:
1. Player name
2. Player region (power rankings points are isolated to each region)
3. Set of events the player placed in

Because all tournament results are stored in Cargo, (3) is naturally very easily determined from (1) automatically. Because players typically play in their own region, we assign to them the region of their first tournament participation, which can also be looked up via Cargo. That takes care of (2). Every day at about 5am Pacific time, a script runs on a server I have and creates every player page that has power rankings points associated to it but hasn't been created yet. This automatic creation is what I was planning to start to do for Leaguepedia as well.

What you might guess from this is that we don't really care about notability guidelines on the *Fortnite* wiki (I use "we" in the general sense - I'm not a *Fortnite* admin, I just write code). Of course this automated system doesn't capture support staff, and support staff pages are generally the most commonly-asked-about when it comes to notability, but for players, if you place in an event and get Power Rankings points, you get a page.

Over the roughly 9 months that this system has been live, it's gone pretty well. There is a huge volume of requests for support from the staff of the wiki, but the primary reason for that would be unlikely to be an issue on the Leaguepedia, because players rename in *League of Legends* significantly less than they do in *Fortnite*, and our *League* data is not raw data from live servers (usually...), so the problem of associating two names of the same player would be significantly alleviated.

From a database point of view, there is some concern about accumulating too much data over time, but I have a second script which flags players as "inactive" if they haven't had a placement in the past 6 months, and have 2 or fewer lifetime placements (this is a huge percentage of all pages). On the *League* wiki, my estimate was that we would create somewhere between 10,000 and 30,000 player pages. This is a lot, and it would make database table rebuilds take approximately 500 [Bearimies](https://thegoodplace.fandom.com/wiki/Jeremy_Bearimy_Timeline) (*The Good Place* spoiler warning!) to complete, but 50,000 rows in a table is not that bad.

So, just based on the experiment of the *Fortnite* wiki, it does seem possible to create a page for every *League of Legends* player who played in an event we have recorded on the wiki.

## Okay...but why?

Why would we ever want this? Good question. There's a few reasons.

To me, the most compelling reason is that if we have data, I want a place to display it. We have the ability to query and collate all results of any player in any event - I want a way to show this data grouped by players. The most natural place to show the results of a query like this is of course a page for the player in question. (There are other possibilities though - think [query pages](https://lol.gamepedia.com/index.php?title=Special:RunQuery/RegionHeadToHead&pfRunQueryFormName=RegionHeadToHead&RHTH[1]=na&RHTH[2]=eu&pf_free_text=&wpRunQuery=Run+query).)

Another reason is simply that players tend to want to have pages - and this would be a low-effort way to create a lot of wanted pages (aside from, you know, the time it would take me to build this entire thing). This is somewhat of a double-edged sword though - if we are making pages for everyone, this takes away a lot of the meaning of meeting our notability guidelines, and this issue was one of the reasons that I'm writing this blog post instead of a fake April Fools Day announcement.

But there's one reason above all others, which is that our new roster changes code (which is *incredbly* exciting, btw) isn't really set up to support players without pages. There's a rule that I both made up and try to follow, which I'm calling the "0th normal form" of having a relational database attached to a wiki. I'm eventually going to write an entire blog post about this one topic, but the general idea is that if you have data about an entity (a player or a tournament or a roster change), the entity's identifier better tell you *exactly* where on the wiki to find that piece of information So player information goes on a player page, tournament information goes on a tournament page, and roster change information goes on a page associated with the date of the roster change.

As a result of this rule, if I have data about a player, it's gotta be on the player's page, and the player's page has to exist. Data could include IRL name, residency, country, etc - in the old system these data points would be entered by hand (or copy-pasted) on every team page and every tournament rosters page. In the new system, no data is allowed to be entered by hand on the team page, it all has to come from player pages. Which means...it's not possible to show a player on a team page unless we have a page for the player, or, in other words, every player who shows up on any team page, in turn needs to have their own page.

Currently, we're at a sort of in-between state: One database table, `ListplayerCurrent`, has its data stored from team pages via the template `Listplayer/Current`. This is the old table (I would never give a table this nonspecific of a name anymore). The new table is called `Tenures` (actually there are two tables: `Tenures` stores constant-status tenures, so when a player changes role, the old tenure ends and a new one begins; a second table, `TenuresUnbroken`, stores complete tenures without breaking them up into pieces when statuses change), and it is stored from player pages. There's a flag in the `Teams` table that editors can set to tell the wiki "pull data about this team's current players from `Tenures` instead of `ListplayerCurrent`".

This situation, as convoluted as it sounds, technically does not violate the 0th normal form because an entity in this table is a player+team, and in one table it can be always found on the team page, and in the other it can be always found on the player page. But what I *can't* do is migrate current team pages to use `Tenures` and have `Tenures` sometimes be stored to from player pages and sometimes from team pages; that would be a blatant violation.

## Okay, so why not?

As I mentioned before, there is a desire not to have a large number of incomplete, automatically-generated player pages among the carefully-curated pages we have. This both decreases the average quality of a wiki page and also cheapens the value of having a page in the first place. One possible solution we have discussed is to have automatically-generated pages have a significantly different visual style from regular pages. For example, we could simply not include an infobox anywhere on the page. Or if we have an infobox, it wouldn't have an image, and its styling could just be a thin black border instead of being colorful. If we do go ahead with the Snap in the future, we will almost certainly incorporate something along these lines.

The second reason is maybe more boring, but it was also the more decisive. The reason is, "Becky said it would probably be bad for SEO." Becky is Fandom's SEO expert, and if she says something is bad for SEO, we just...don't do it. (Usually.) (Okay, sometimes.)

## What now, then?

Who knows! The option that I've been toying around with the most is to create pages "Data:PlayerName" that store data, and provide a one-click button to delete a data page and turn it into a live player page, so that we can enter information into a non-user-facing location without actually creating a page. Disadvantages include that this is still technically a 0NF violation - the entity doesn't determine if it's stored on a "real" page or a data page - albeit now a very, very minor violation: it's easy to create automated links to data-namespace pages from the main namespace, and in fact we already do this for tournament data pages.

Another disadvantage is that this is a *lot* more coding intensive - I need to set up easy workflows for creating a data page instead of a regular page, navigating to and from this data page, and moving the data page live if and when the time is right; and additionally, this needs to be done without incorporating the data page into the standard navigation for the player, because the page doesn't actually exist yet.

Also, of course this solution doesn't create the location for accumulating tournament results in any way, since there is no live page period.

Another option is to partially snap - instead of creating tens of thousands of pages, maybe we only create a couple thousand. If we limit to players linked to from team pages who also have additional pieces of data beyond ingame ID, maybe we can keep our situation relatively well controlled. [Update - As of June 9, this is what we are doing! Snapped pages are being created for players mentioned in team pages, but not for players only mentioned in tournament pages.]

Whatever we do, this has been a very fun scenario to consider over the past several months, and I think the final product is guaranteed to be exciting! (At least, if you are excited by data management it will be...)