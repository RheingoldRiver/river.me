---
title: "Determining team members from roster change data: A Leaguepedia case study"
date: 2020-02-18T23:55:49Z
draft: false
tags:
    - mediawiki
    - cargo
    - leaguepedia
summary: A case study in complexity management using Cargo with MediaWiki to track League of Legends esports roster changes.
---

The goal: Create a structure of database tables that supports having a "single source of truth" for all data relating to players' tenures on teams: Players' team histories, all former members of any team, reports of changes made during off-season, players' current teams, teams' current players, news items about players, news items about teams, and anything else that can be determined using this data.

Actually, I skipped over some of the things that made this difficult in that list (you probably skimmed it anyway).

I've [previously](https://river.me/blog/new-roster-changes/) chronicled the process to solve this problem, but that was more of a brief overview of everything rather than an indepth look at a single part. In this part I'm only going to discuss the Team Members part - which in my previous post I mentioned I hadn't yet decided how to do. Well, I've written the code now, and I'm completely convinced I made the right decision. So here's what I did.

(This post isn't really meant to be a tutorial, just a discussion of some of the things I took into account when designing this system.)

The dilemma was, in short, do I create an additional database storage dependency arrow from team pages to player pages, or do I do a ton of recalculation on the team page to determine members of the team directly from raw change data? The problem wasn't that both of these options seemed attractive; rather, they both seemed completely untenable.

It's *very* easy to fall into a trap of adding infinite dependency arrows between all pages in a situation like this - when you have a single source of truth, by definition you start out with only one model of your data to access. Any other model requires you either to query and calculate it from scratch, or to add a dependency arrow where one page has to be blank edited for another to have correct data. So as a result, I'm *extremely* reluctant ever to go through the steps of query-calculate-store on a page and have dependencies on that following store. And despite this reluctance, I still have to do it on occasion, which means that I'm even *more* reluctant to do the same later (aka now).

But on the other hand, I was legitimately concerned about running into physical limits in what Lua is allowed to execute each page load. Here's why: To calculate the players who played on a team, I only needed to examine changes involving that team. But we don't display ONLY the names of players: There's also player residencies, which can change over time, and the **next team the player joined after leaving this team**. (And for good measure, players can and often do change their names, so I need to retrieve some historical data about players renaming as well.)

Ok so you need some historical data about players. What's so bad about that? Well....There's a lot of data to be forced to query and process. What I want is a query that gives me one row per player with all of my needed ancillary data. This is not going to be possible. Because I'm querying with a team in the `where` condition, my best bet for getting any player-but-not-team related data will need a `join`. And the best bet for a `join` is another copy of the news-by-date table where I restrict the copy to be strictly later date than the original. But this is now a one-to-many join in a table that has a huge number of rows. And there's not really any way to restrict this query:
* There's multiple players, so the strategy of setting some limit doesn't work (e.g. order by date asc, where date is at least team activity date, limit 1).
* Even if it did, I actually need more than one row of data because some points will be team changes, some renames, some nationalities, and I need the first of *each* of these, and by necessity, they are recorded in the same single News table to give an ordering by date.
* And in fact I don't even want the immediate next row in a lot of cases; for example, the next row about a player could be that they left a team they had *already* been on, prior to this team; and then we wouldn't want to fill with that line, we'd want the next one after that.
* I can't do one query per player for obvious performance reasons.

So I'm joining a second copy of `NewsItems` on the sole condition that the date is later than the player leaving the team in question. This condition DOES restrict my data set a bit, but not that much. If I were to just analyze every line from every player, I have `O(n*m)` data points, where there are `n` total players who have ever been on the team, and each player has an average of `m` total transfers in their history. When I restrict to only transfers occurring after the tenure on this team, then all I've done is restrict to (on average) half of each player's career - so it's `O(n*m/2)` which is still `O(n*m)`.

Is `O(n*m)` actually quadratic? After all, if each player is on exactly 10 teams prior to retiring, this is just linear proportional to age of team. I argue it is quadratic, though - even if `m` is uncorrelated with age of team, both `n` and `m` are correlated to another variable - *age of the wiki*. We are not yet at the point where there's an effective cap on the number of teams a player joins in their career, and while there certainly are a lot of teams that are created and almost immediately disband, teams in primary leagues are getting more and more stable with franchising. While you could argue that we are maybe close to a place where the *average* case `m` is a constant, we care a lot more about the worst case than an average case, and in worst case there's still a large number of players still remaining active for close to the entire life of the wiki (and certainly for the life of "modern" teams).

Ok well, do we actually care about this being quadratic? Here's where I admit I violated rule 0 of making a decision for performance reasons....I didn't test the performance. But a quick order of magnitude calculation told me that currently, there should be about 50-75 players on a "typical" long-lasting team's history, with each player having on average about 10 teams in their history. Since a team tenure actually has 2 lines (a join and a leave), this is order of magnitude 1000-1500 lines of Cargo. Say in 5 years, both `m` and `n` have doubled. Then we're looking at 4000-6000 rows. The maximum rows Cargo can return in one query is 5000. Again, maybe okay - certainly doing 2 queries instead of 1 isn't the end of the world. But this seems pretty sketch.

But let's get to the real reason I didn't want to do this. It's the same reason I didn't test the performance.

I REALLY DID NOT WANT TO CODE THIS SHIT.

So the code that determines tenures is pretty much black magic. Each change has a direction associated with it, and I need to pair up compatible rows while skipping every redundant row (e.g. a player could be added to the contract database on Thursday and announced on Monday - both of these would be news lines, but I only want to pick the first one). So there's a lookup of players to relevant lines, and each line has data about the role, role modifier, etc., and whether it's "completed" (is there both a leave and a join). At the same time, we're adding ancillary data do each row - did the player recently change name? What about residency? Better update every previous line as needed.

All of this also has to be used in two totally different contexts - one of these is leave-to-join (offseason change), and the other is join-to-leave (team history). I made the maybe-questionable decision to subclass here, because there's a huge amount of code reuse between cases. And I just don't want to deal with this in an even more-complicated case than I currently have. I tend to respect when I really don't want to code something - usually it means the complexity is just not worth it.

In contrast, I prototyped the option that creates the extra dependency in about 30 minutes. But is that code simplicity actually worth the permanent dependency? Honestly, yeah. Given that I also have performance concerns about using the other method (though again you should NEVER make a performance argument without testing), this is worth it.

There were a couple other points that solidified my opinion:
* There's already a dependency on player pages from team pages, the "Player-League History" section. In fact, the reason for that dependency is almost identical to the reason for this one - there would be an insanely huge amount of data needed to be queried with a lot of complexity on the team page, but if stored on the player page and then queried, the entire system is very simple (other than the dependency of course).
* One of the reasons I was unhappy about the Team -> Player dependency is that I didn't want Player -> Team -> Player, where navboxes on player pages are queried from current team members on team pages. But I can move the data source for these to be the Tenures table on player pages, so I won't have this cycle once this system is completely rolled out.
* I am storing the Tenures table no matter what, because I want to be able to do queries along the lines of "longest time a player spent unbroken on the same team" - so since this will become a table that needs to be updated no matter what, the dependency chain isn't so bad.
* I have a gadget that editors are required to run after saving any news item, and I can impose an order in which this gadget blank edits pages (currently it does everything at once), so unless editors don't use the gadget (in which case nothing gets updated regardless), there shouldn't actually be any cases of pages updated out of order or incompletely.

In the end, the code for team members turned out pretty clean, and I was able to focus on adding cool features (including a pop-up button that shows a player's complete team history, loaded lazily).