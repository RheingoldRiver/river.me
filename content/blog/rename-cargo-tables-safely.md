---
title: "How to rename Cargo tables safely"
date: 2020-08-11T01:08:14Z
draft: false
tags:
    - cargo
    - leaguepedia
    - best_practices
summary: Cargo actually provides a way to cleanly rename tables without any "downtime," but it's not obvious or documented how to do this.
description: Cargo actually provides a way to cleanly rename tables without any "downtime," but it's not obvious or documented how to do this.
---
Renaming Cargo tables is unavoidable in the lifetime of a wiki. But it is potentially an extremely disruptive process - every single store and every single query must be changed, and if you miss one, the effects can be disastrous. Because the `cargoquery` action is available through the Mediawiki api, the effects of a table rename can ripple even farther to third-party users who are querying your data from an external script or program, and you may want to give a grace period of a week or even a month for users to adjust to the rename.

It turns out that there's actually a very simple procedure for renaming a Cargo table that allows for a zero-downtime transition period for both yourself as a wiki developer and your API users, but as far as I know it's not documented anywhere (yet), and I think it's actually included in the extension unintentionally. So in this article I will present reasons for renaming tables, (what I'm guessing are) commonly-used and incorrect strategies for renaming tables, and finally, the proper method for renaming tables. If you are only here for the last part, I won't blame you [for skipping to the end](#the-right-way-to-rename-a-cargo-table) :)

## Why would you rename a Cargo table?
The most obvious reason for renaming a Cargo table is perhaps that you messed up - you didn't write a thorough style guide at the start of building your table schema and it's starting to haunt you, or you made a typo, or you forgot what your style guide says. But even barring human error, there's several reasons you could want to rename a table.
* The real world changes - maybe the "Global Contract Database" (this is an [actual thing](https://lol.gamepedia.com/Archive:Global_Contract_Database)) is now called the "International Contract Database" (I made this up) and "GCD" needs to be renamed to "ICD." 
* Requirements change - your table was called "Employees" but you've gotten word that it's gotta include contractor data too, and your manager is insisting that you rename it to "Personnel."
* Conventions change - maybe you overhaul your style guide after four years of experience. Is a one-time disruption worth it to improve consistency in all of your naming? If the disruption isn't that bad, maybe it is.
  
## How not to rename a Cargo table
The first three methods I outline here are three variations on a similar theme, then the fourth method is a much higher-effort option that you could take if you needed long-term stability during your maintenance period.

While these scenarios are only provided for the purpose of illustration, do keep in mind that site notices are a valuable tool whenever you do need an extended maintenance that could create broken or incomplete data - use them liberally to inform your users about ongoing development projects!
### Downtime, with errors everywhere
This is probably the easiest way to rename a table. If you're in the early stages of using your table - especially if you don't have any queries yet! - this is also totally acceptable. If your wiki is low enough traffic, or a personal project, this might also be fine.
#### The method
1. Prepare all of your queries to use the new table but don't save anything yet
2. Prepare the stores to use the new table but don't save anything yet
3. Delete the old table
4. Change the declaration to use the new table, and save
5. Create the new table
6. Change the attaches to use the new table, if there are any
7. Save the stores
8. Save all of your queries
9. Hope the jobqueue catches up really fast because you have errors all over the place until it does

#### Why it's not a great idea
Everything is broken for a while, and there's no time for any third-party developers to deploy new code before their old code breaks, if anyone is using your api

### Incomplete data
I personally prefer to have things be obviously broken than to be silently incomplete. Depending on your particular use case, though, the opposite might be the case - if your data isn't particularly precise, but your users are very skittish, then you may prefer to avoid errors or empty fields, even if it means potentially wrong or incomplete info. The "empty stuff instead of errors" can be a nice compromise between this option and the first, but it's more work than either of them.
#### The method
1. Change your declaration and stores to use the new name
2. Create your new table
3. Update your queries ASAP, before too much data disappears from your old table (due to page edits)
4. Delete your old table
#### Why it's not a great idea
This method is the most similar in practice to [the right way](#the-right-way-to-rename-a-cargo-table) - but the disclaimer in step 3 that data can disappear from your old table can be a pretty big deal, especially if you want to give a several-day-long grace period to third-party users. Data will slowly (or quickly) disappear from the old table, and queries still using the old table will get more and more out of date, with no indication that they are doing so.

### Downtime, with empty stuff
A bit more work than having errors everywhere or incomplete data, if you don't want to scare your users or mislead them with incomplete data.
#### The method
1. Prepare all of your queries to use the new table but copy-paste the code into an external text editor instead
2. Comment out all your queries and save
3. Same as the first method from here on out, except there won't be any errors, just empty stuff

Note: The copy-paste into an external text editor is to avoid edit conflicts.
#### Why it's not a great idea
It's still not a great experience for users because their data disappears. This can also break literally everything if you ever have store-query-store patterns. It's also more work than the earlier methods.

### Ctrl C, Ctrl V
This is a way to have your old table running in parallel with your new table, but it has a lot of downsides, as you will see after the method is presented.
#### The Method
1. Create a new template with a copy-paste of the original table's declaration, except with the new name
2. Copy-paste the `#cargo_store` block of code in every template that stores to the table
3. Update all of the queries to use the new table
4. Delete the old stores
5. Delete the old table
6. Delete the old declaration, and move the new declaration into the storing template. Delete the temporary template you made in the first step.

#### Why it's not a great idea
First of all, the copy-paste of the `#cargo_store` is MUCH easier said than done in a lot of cases. MediaWiki templates tend to have a lot of side effects due to the prevalence of the `#var` parser function and the fact that all variables it creates are global. Even in Lua, it's not unlikely that storing a second copy of the every row could get pretty complicated.

Second, from a performance standpoint this is a NIGHTMARE. If your wiki is very lightweight, maybe this doesn't matter, but you're literally doubling the number of database writes to this table - even if it's temporary, that can end up a strain on resources.

Third, it requires you to edit each template not just once, but twice, which opens up twice the opportunities for you to mess up. And on top of this, the edits are significantly more complicated than what they should be, which is a single word change (the name of the table).

Finally, you can't `#cargo_attach` more than one table to the same template. So if more than one template is storing to this table, and you are already attaching the old table, you will need to use the [multiple-table attach workarournd that I documented on the Gamepedia Help wiki](https://help.gamepedia.com/Extension:Cargo/attaching_tables).

## The right way to rename a Cargo table
We can take advantage of the replacement table mechanism to maintain a read-only copy of the original table until we've finished transitioning to the new table:
1. Create a replacement table for your old table. The original version of your old table is now read-only, forever.
2. Change your declaration and store(s) (and attach(es), if applicable) to use the new name
3. Create your new table
4. Update your queries to use the new table
5. At your convenience, delete the replacement and then original of your old table.

## Conclusion
The reason this article has a `leaguepedia` tag is that in May this year I used this method for renaming three core tables - `ScoreboardPlayer` became `ScoreboardPlayers`, `ScoreboardTeam` became `ScoreboardTeams`, and `ScoreboardGame` became `ScoreboardGames`. I gave a one-week-long period of time when the old tables existed in a read-only state for people to change their application code to use the new tables; if I were to redo the process today, I'd probably try to give a longer grace period of two weeks or even a month.

The original names were legacy holdovers from back when we used Semantic MediaWiki, even before I was part of the wiki, and they were the only tables in my entire schema not to use plural names. So I decided that it was worth a one-time disruption to rename the tables for consistency, gave notice, and went through with the renames!