---
title: "Compound Role"
date: 2020-07-20T16:50:18Z
draft: true
tags:
    - leaguepedia
summary: A bunch of things I did wrong because learning is fun!
---

## Background
I have a bunch of infrastructure on my wiki related to taking user input, validating it against a set of recognized keys, returning a set of values, and then printing an appropriate display along with maybe an image (which may be a sprite). For example if you type `a` for a role, and the "length" to display is `store`, the wiki will print `Bot`. If the "display" is the `store` key, then the wiki will "print" `AD Carry` (we use the historical name of the position in all my internal database tables). There's a bunch of these modules - one for champions, one for teams, one for countries/flags/nationalities, even one for preferred gender pronouns.

An object-oriented approach to these display modules is appropriate since they are able to inherit a significant amount of behavior but do require some degree of polymorphism - for example, roles require a "modifier" - a player could be a "substitute" or "trainee," a concept that doesn't exist for a champion. If the input a user gives isn't recognized, the default behavior also differs - for a role input, we might assume this is an unrecognized "support staff" role, whereas for a champion or country we'll error.

Until recently, I haven't used an object-oriented approach, though, and it will probably be at least a year, if not longer, before all of my code is migrated to the new system I'm currently writing. The approach has evolved over time; originally there was no Lua involved, and there were a bunch of `#switch` statements in wikitext code; initial implementations in Lua were 1:1 translations of this approach, which I slowly morphed over time. Currently I have an `EntityAbstract` which I subclass.

Collectively I refer to these modules as "markup entities" or just "entities," and Role is by far the most complicated because a role isn't actually a well-defined entity, but in the past it's always been treated as one, with worse and worse workarounds over the years.

## Complications of Role
![Screenshot of a player's team history showing roles with modifier](/images/compound-role/role-with-sub.png)
Role has several different contexts:
* As a well-defined set of ingame positions: Top, Jungle, Mid, AD Carry/Marksman/Bot, Support (the name of the 4th position has changed a couple times) - this is used in ingame data like pick-ban order, scoreboards, stats, etc
* As a set of ingame positions with optionally a modifier like "substitute" or "trainee" - this is used in team roster lists, etc - above you can see a screenshot of a top lane icon with an `S` superimposed for "Substitute."
* As a poorly-defined position that could be either an ingame position or any support staff position such as "Coach" or "General Manager" or "Sports Psychologist" or "Chef" - this is used on player/support-staff pages and in organization member lists, in roster changes, etc

The situation is more complicated than it seems, however. In the last scenario, a player could have more than one role at the same time, as shown in the following screenshot:
![Screenshot showing a player with both ingame and staff icons at the same time](/images/compound-role/player-and-staff-roles.png)
This screenshot would not have been possible a month ago; instead of recognizing that "support;manager" was in fact a list of two roles (or a "Compound Role" as the title of this article might suggest), it only knew how to replace the `;` with a `/` to make it slightly more visually appealing (I did mention shitty workarounds) and then treat it as "random poorly-defined non-player role."

(It was a bit worse - remember in the introduction I said that my internal database normalizes the `bot lane` role as `AD Carry`? Yeah so it was printing as `AD Carry/Manager` even though this term is never supposed to be user-facing. Whoops.)

## CompoundRole
As you may have guessed, one of the many major changes I made to the system for `Role` specifically was to add a new object: `CompoundRole`. This object is basically just a list of `Role`s, but its introduction means that any time I'm printing or dealing with something that might be a list of roles, I don't have to worry about whether the thing is a list or not; I can just instantiate a `CompoundRole` instead of a `Role` and be done with it.

`CompoundRole` subclasses `CompoundEntityAbstract` and inherits methods like `:images()`, `:flairs()`, `:exists()`, etc. 