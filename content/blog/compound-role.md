---
title: "CompoundRole"
date: 2020-08-26T16:50:18Z
draft: false
tags:
    - leaguepedia
    - lua
summary: A bunch of things I did wrong because learning is fun!
description: A bunch of things I did wrong because learning is fun!
---
This post doesn't have a particular goal in mind - I just wanted to write about the evolution of a Lua module that went through a large number of iterations in July.
## Background
I have a bunch of infrastructure on my wiki related to taking user input, validating it against a set of recognized keys, returning a set of values, and then printing an appropriate display along with maybe an image (which may be either a sprite or a file).

{{% leaguepedia_citation %}}

Here's a snippet from the [settings file for displaying roles](https://lol.gamepedia.com/Module:Rolenames):
```lua
	['4'] = 'ad carry',
	a = "ad carry",
	ad = "ad carry",
	adc = "ad carry",
	marksman = "ad carry",
	b = "ad carry",
	bot = "ad carry",
	["bot laner"] = "ad carry",
	["ad carry"] = { short = "Bot", name = "Bot", role = "Bot Laner", sentence = "bot laner", prep = "for", article = "a", sortnumber = 4, portal = "Bot", },
```
As this example shows, if you type `b` for a role, and the "length" to display is `short`, the wiki will print `Bot`. If the "length" is the `role` key, then the wiki will print `Bot Laner`. Up until recently, we had a `store` "length" used for Cargo tables, which used the legacy name `AD Carry` - in the time since I started writing this post and when I finished it, I changed everything to use `Bot` instead, even internally. This is relevant later.

There's [a bunch of these modules](https://lol.gamepedia.com/Category:Entity_Name_Modules) - one for [champions](https://lol.gamepedia.com/Module:Championnames), one for [teams](https://lol.gamepedia.com/Module:Teamnames), one for [countries/flags/nationalities](https://lol.gamepedia.com/Module:Countrynames), even one for [gender pronouns](https://lol.gamepedia.com/Module:Pronounnames).

An object-oriented approach to these display modules is appropriate since they are able to inherit a significant amount of behavior but do require some degree of polymorphism - for example, roles require a "modifier" - a player could be a "substitute bot laner" or "trainee bot laner," a concept that doesn't exist for a champion (a playable character in the game *League of Legends*).

If the input a user gives isn't recognized, the default behavior also differs - for a role input, we might assume this is an unrecognized "support staff" role, whereas for a champion or country we'll error, assuming that our list of recognized inputs is definitive.

Until very recently, I hadn't used an object-oriented approach, though, and as of writing this article I'm still not completely done migrating to using one - my [Team display module](https://lol.gamepedia.com/Module:Team) is still using an older method, though [everything else is now migrated](https://lol.gamepedia.com/Category:Entities). The approach has evolved over time; originally there was no Lua involved, and there were a bunch of `#switch` statements in wikitext code; initial implementations in Lua were 1:1 translations of this approach, which I slowly morphed over time. Currently I have an [EntityAbstract](https://lol.gamepedia.com/Module:EntityAbstract) which I subclass.

Collectively I refer to these modules as "markup entities" or just "entities," and Role is by far the most complicated because a role isn't actually a well-defined entity, but in the past it's always been treated as one, with worse and worse workarounds over the years.

## Complications of Role
![Screenshot of a player's team history showing roles with modifier](/images/compound-role/role-with-sub.png)
Role has several meanings depending on context:
* As a well-defined set of ingame positions: Top, Jungle, Mid, AD Carry/Marksman/Bot, Support (the name of the 4th position has changed a couple times) - this is used in ingame data like pick-ban order, scoreboards, stats, etc
* As a set of ingame positions with optionally a modifier like "substitute" or "trainee" - this is used in team roster lists, etc - above you can see a screenshot of a top lane icon with an `S` superimposed for "Substitute."
* As a poorly-defined position that could be either an ingame position or any support staff position such as "Coach" or "General Manager" or "Sports Psychologist" or even "Chef" - this is used on player/support-staff pages and in organization member lists, in roster changes, etc

The situation is more complicated than it seems, however. In the last scenario, a player could have more than one role at the same time, as shown in the following screenshot:
![Screenshot showing a player with both ingame and staff icons at the same time](/images/compound-role/player-and-staff-roles.png)
This screenshot would not have been possible two months ago; instead of recognizing that "support;manager" was in fact a list of two roles (or a "Compound Role" as the title of this article might suggest), it only knew how to replace the `;` with a `/` to make it slightly more visually appealing (I did mention shitty workarounds) and then treat it as "random poorly-defined non-player role."

(It was a bit worse - remember in the background I said that my internal database used to normalize the `bot lane` role as `AD Carry`? Yeah so it was printing as `AD Carry/Manager` even though this term was never supposed to be user-facing. Whoops.)

## CompoundRole
As you may have guessed, one of the many major changes I made to the system for `Role` specifically was to add a new type: `CompoundRole`. This type is basically just a list of `Role`s, but its introduction means that any time I'm printing or dealing with something that might be a list of roles, I don't have to worry about whether the thing is a list or not; I can just instantiate a `CompoundRole` instead of a `Role` and be done with it.

`CompoundRole` subclasses `CompoundEntityAbstract` and inherits methods like `:images()`, `:flairs()`, `:exists()`, etc.

### Limitations
There's one major limitation to the `CompoundRole` behavior I get for "free" from `CompoundEntityAbstract`: Frequently, I require the ability first to filter roles by whether they are ingame positions or not and then apply some method to them. This filtering behavior is completely unique to role and has no analog in any similar Entity. How to deal with this? I went through several iterations.

## Adding to constructor

For a time, my `init` looked roughly like this:

```lua
function p:init(str, opts)
	if not opts then opts = {} end
	opts.sep = opts.sep or ';'
	self:super('init', str, opts)
	if self.is_nil and opts.modifier then
		-- case when we have a modifier but no string (e.g. solely sub or trainee)
		self.objs = { self.Entity(nil, opts) }
		self.is_nil = false
	end
	if self.is_nil then return end
	
	self.ingame = self:filterIngameRoles(true)
	self.staff = self:filterIngameRoles(false)
	self.hasIngame = #self.ingame > 0
	self.hasStaff = #self.staff > 0
	self.ingameOnly = not self.hasStaff
	self.staffOnly = not self.hasStaff
end

function p:filterIngameRoles(isIngame)
	if self.is_nil then return end
	local filteredRoles = {}
	for _, role in ipairs(self) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end
```

And then I had some other methods like this:

```lua
function p:ingameImages(opts)
	if self.is_nil then return end
	if not opts then opts = {} end
	local tbl = {}
	for _, obj in ipairs(self.ingame) do
		tbl[#tbl+1] = obj:image(opts)
	end
	return util_table.concat(tbl, opts.sep or '')
end

function p:ingameNames(length, opts)
	if self.is_nil then return end
	if not opts then opts = {} end
	local tbl = {}
	for _, obj in ipairs(self.ingame) do
		tbl[#tbl+1] = obj:name(length, opts)
	end
	return util_table.concat(tbl, opts.sep)
end

function p:staffNames(length, opts)
	if self.is_nil then return end
	if not opts then opts = {} end
	local tbl = {}
	for _, obj in ipairs(self.staff) do
		tbl[#tbl+1] = obj:name(length, opts)
	end
	return util_table.concat(tbl, opts.sep)
end
```

But I realized, this kind of sucks, I already have methods called `:images()` and `:names()`, wouldn't it be nicer if I could call something like `.staff.:names()` instead of having to write separate methods for each way to filter like this? I should be creating new objects when I filter to staff roles or ingame roles, not just lists that I'm unable to work with.

## RoleSet

So my next solution was to introduce a second new type, called `RoleSet`, that could do precisely this:

```lua
local CompoundRole = require('Module:CompoundRole')
local LCS = require('Module:LuaClassSystem').class
local p = LCS()
function p:init(strs, opts)
	self.all = CompoundRole(strs, opts)
	local opts2 = mw.clone(opts)
	opts2.alreadyCast = true
	self.ingame = CompoundRole(h.filterIngameRoles(self.all, true), opts2)
	self.staff = CompoundRole(h.filterIngameRoles(self.all, false), opts2)
	
	self.hasIngame = self.ingame:exists()
	self.hasStaff = self.staff:exists()
	self.isIngameOnly = not self.staff:exists()
	self.isStaffOnly = not self.ingame:exists()
end

function h.filterIngameRoles(objs, isIngame)
	local filteredRoles = {}
	for _, role in ipairs(objs) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end

function p:tostring()
	return self.all:tostring()
end

function p:__ipairs()
	return self.all.__ipairs()
end

return p
```
This object gave me a list of all roles, a list of ingame roles, and a list of staff roles.
### Problems
There were just too many cases of not-type-safe code, and using this approach would have required too much refactoring to make work. For example, team rosters are certainly ingame-only CompoundRoles, but they can also store to news. If news expects a `RoleSet` but `TeamRoster` is dealing with `CompoundRole` then I needed a different constructor that could create a `RoleSet` from a `CompoundRole` instead of from a string, or I needed to preserve my string args and create first a `CompoundRole` in my TeamRoster code and later a `RoleSet` in my News code, or just deal with RoleSets always, but then I'm *always* dealing with RoleSets, and that's annoying, and the whole thing was annoying, and the entire point was to not be annoying.

This also seemed like a problem to serialize and deserialize from Cargo: Was I going to make everything into a `RoleSet` when I store in Cargo? If so, that's a rather complicated serialize-deserialize process, since I need at least two distinct separators (one to separate roles from each other, and another to separate ingame from not-ingame). And if I'm not doing this then I'm not preserving types before and after queries, and that sucks, and I really just didn't want this extra type to deal with.

## Back to the constructor

The next thing I tried was something like this:

```lua
function p:init(str, opts)
	if not opts then opts = {} end
	opts.sep = opts.sep or ';'
	self:super('init', str, opts)
	if self.is_nil and opts.modifier then
		-- case when we have a modifier but no string (e.g. solely sub or trainee)
		self.objs = { self.Entity(nil, opts) }
		self.is_nil = false
	end
	if self.is_nil then return end
	
	if not opts.alreadyCast then
		local opts2 = mw.clone(opts)
		opts2.alreadyCast = true
		self.ingame = p(h.filterIngameRoles(self, true), opts2)
		self.staff = p(h.filterIngameRoles(self, false), opts2)
		
		self.hasIngame = self.ingame:exists()
		self.hasStaff = self.staff:exists()
		self.isIngameOnly = not self.staff:exists()
		self.isStaffOnly = not self.ingame:exists()
	end
end

function h.filterIngameRoles(objs, isIngame)
	local filteredRoles = {}
	for _, role in ipairs(objs) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end
```
The `opts.alreadyCast` stops this from being an infinite loop (whoops, I forgot to do that at first), and now I can call things like `role.ingame:images()`. But in addition to just feeling kind of clunky and gross, this is also a lot of extra computation that I frequently don't need to do - probably 80% of the time that I instantiate a `CompoundRole`, I don't actually need to do any ingame/staff filtering.

## CompoundRoleSet
So then I thought, okay, what if I go back to having a second type, but this time, what if I make it subclass `CompoundRole`? That fixes some of the problems I had before, because now it inherits the methods it needs. This way if I don't need to create the ingame/staff filters I won't, but I still keep the methods I need.

```lua
local CompoundRole = require('Module:CompoundRole')

local CompoundRoleSet = CompoundRole:extends()

CompoundRoleSet.Entity = require('Module:Role2')

local h = {}
function CompoundRoleSet:init(strs, opts)
	self:super('init', strs, opts)
	local opts2 = mw.clone(opts)
	opts2.alreadyCast = true
	self.ingame = CompoundRole(h.filterIngameRoles(self, true), opts2)
	self.staff = CompoundRole(h.filterIngameRoles(self, false), opts2)
	
	self.hasIngame = self.ingame:exists()
	self.hasStaff = self.staff:exists()
	self.isIngameOnly = not self.staff:exists()
	self.isStaffOnly = not self.ingame:exists()
end

function h.filterIngameRoles(objs, isIngame)
	local filteredRoles = {}
	for _, role in ipairs(objs) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end

return CompoundRoleSet
```
I still didn't like this.
## Active filtering methods on CompoundRole
Finally I changed to the method (haha) that I'm using now, and this has been in place for over a month, and I've successfully migrated all of my code to use it, so I think it's safe to say it's final: I deleted `CompoundRoleSet` and went back to having the filters in `CompoundRole`, but now they are available as methods, not part of the `init`:

```lua
function p:ingame()
	return self:getFilterKey('ingame')
end

function p:staff()
	return self:getFilterKey('staff')
end

function p:hasIngame()
	return self:getFilterKey('hasIngame')
end

function p:hasStaff()
	return self:getFilterKey('hasStaff')
end

function p:isIngameOnly()
	return self:getFilterKey('isIngameOnly')
end

function p:isStaffOnly()
	return self:getFilterKey('isStaffOnly')
end

function p:getFilterKey(key)
	if self.isFiltered then return self.filters[key] end
	self:getFilters()
	return self.filters[key]
end

function p:getFilters()
	optsIngame = mw.clone(self.opts)
	optsStaff = mw.clone(self.opts)
	optsStaff.modifier = nil
	optsStaff.trainee = nil
	optsStaff.Trainee = nil
	optsStaff.sub = nil
	optsStaff.Sub = nil
	optsIngame.alreadyCast = true
	optsStaff.alreadyCast = true
	local ingame = p(h.filterIngameRoles(self, true), optsIngame)
	local staff = p(h.filterIngameRoles(self, false), optsStaff)
	self.filters = {
		ingame = ingame,
		staff = staff,
		hasIngame = ingame:exists(),
		hasStaff = staff:exists(),
		isIngameOnly = not staff:exists(),
		isStaffOnly = not ingame:exists(),
	}
	self.isFiltered = true
end

function h.filterIngameRoles(objs, isIngame)
	local filteredRoles = {}
	for _, role in ipairs(objs) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end
```
Any time a filter is requested, the code first checks to see if a filter has been previously performed, If so, it returns the value computed. If not, then it computes all possible filtered values, saves them all, and returns the one you want. So the syntax looks like `role:ingame():images()`, etc. Infinite looping is "possible" but checks aren't needed since everything needs to be actively called by external logic.

## Renaming
I ended up renaming `CompoundRole` to `RoleList` along the way - at some point I realized that this was a more accurate name for [all of these modules](https://lol.gamepedia.com/Category:Entity_Lists). This blog post is still called `CompoundRole` because for the entirety of the development process discussed here, I was intending to call it `CompoundRole` and also because I wrote most of the post before I renamed it. Also I think `CompoundRole` sounds cooler than `RoleList` even if `RoleList` is a clearer, more descriptive, and better name.

## Current code
### Module:RoleList
Here is the current full code of `Module:RoleList`, and you can find [the module here](https://lol.gamepedia.com/Module:RoleList):

```lua
local util_args = require('Module:ArgsUtil')
local util_html = require("Module:HtmlUtil")
local util_map = require('Module:MapUtil')
local util_table = require("Module:TableUtil")
local util_text = require("Module:TextUtil")
local util_vars = require("Module:VarsUtil")
local i18n = require("Module:I18nUtil")
local lang = mw.getLanguage('en')

local EntityListAbstract = require('Module:EntityListAbstract')

local p = EntityListAbstract:extends()
local h = {}

p.Entity = require('Module:Role')

function p:init(str, opts)
	if not opts then opts = {} end
	self.opts = opts
	opts.sep = opts.sep or ';'
	self:super('init', str, opts)
	if self.is_nil and opts.modifier then
		-- case when we have a modifier but no string (e.g. solely sub or trainee)
		self.objs = { self.Entity(nil, opts) }
		self.is_nil = false
	end
	if self.is_nil then return end

	self.isFiltered = false	
end

function p:names(opts)
	if self.is_nil then return end
	if not opts then opts = {} end
	local opts2 = mw.clone(opts)
	local tbl = {}
	for i, obj in ipairs(self) do
		-- skip the prefix starting with the second item in the list
		-- for example, we want to write Sub/Sup/Assistant Coach
		-- instead of Sub/Sup/Sub/Assistant coach
		if i == 2 then opts2.skip_prefix = true end
		tbl[#tbl+1] = obj:name(opts2)
	end
	return util_table.concat(tbl, opts.sep)
end

function p:sentence()
	if self.is_nil then return end
	local nouns = {}
	local preps = {}
	local phrases = {}
	local prep = self.objs[1]:get('prep')
	
	-- we have to print a preposition next to each word only if
	-- they disagree in the prepositions used, e.g. mid laner for and co-owner of
	-- but mid laner and manager for
	-- so track whether we've changed preposition at all;
	-- if so, then concat and return the phrases
	-- if not then just return the nouns
	local usePreps = false
	for i, obj in ipairs(self) do
		nouns[i] = obj:name{len='sentence'}
		phrases[i] = ('%s %s'):format(obj:name{len='sentence'}, obj:get('prep'))
		if prep ~= obj:get('prep') then
			usePreps = true
		end
	end
	if not usePreps then
		return util_table.printList(nouns) .. ' ' .. prep
	end
	return util_table.printList(phrases)
end

function p:sortnumber()
	if self.is_nil then return end
	return self.objs[1]:sortnumber()
end

function p:ingame()
	return self:getFilterKey('ingame')
end

function p:staff()
	return self:getFilterKey('staff')
end

function p:hasIngame()
	return self:getFilterKey('hasIngame')
end

function p:hasStaff()
	return self:getFilterKey('hasStaff')
end

function p:isIngameOnly()
	return self:getFilterKey('isIngameOnly')
end

function p:isStaffOnly()
	return self:getFilterKey('isStaffOnly')
end

function p:getFilterKey(key)
	if self.isFiltered then return self.filters[key] end
	self:getFilters()
	return self.filters[key]
end

function p:getFilters()
	optsIngame = mw.clone(self.opts)
	optsStaff = mw.clone(self.opts)
	optsStaff.modifier = nil
	optsStaff.trainee = nil
	optsStaff.Trainee = nil
	optsStaff.sub = nil
	optsStaff.Sub = nil
	optsIngame.alreadyCast = true
	optsStaff.alreadyCast = true
	local ingame = p(h.filterIngameRoles(self, true), optsIngame)
	local staff = p(h.filterIngameRoles(self, false), optsStaff)
	self.filters = {
		ingame = ingame,
		staff = staff,
		hasIngame = ingame:exists(),
		hasStaff = staff:exists(),
		isIngameOnly = not staff:exists(),
		isStaffOnly = not ingame:exists(),
	}
	self.isFiltered = true
end

function h.filterIngameRoles(objs, isIngame)
	local filteredRoles = {}
	for _, role in ipairs(objs) do
		if isIngame == role:isIngame() then
			filteredRoles[#filteredRoles+1] = role
		end
	end
	return filteredRoles
end

return p
```
### Module:Role
As a bonus, here's also the current code of `Module:Role`, and [the module is here](https://lol.gamepedia.com/Module:Role):

```lua
local util_args = require('Module:ArgsUtil')
local util_html = require("Module:HtmlUtil")
local util_map = require('Module:MapUtil')
local util_math = require("Module:MathUtil")
local util_table = require("Module:TableUtil")
local util_text = require("Module:TextUtil")
local util_vars = require("Module:VarsUtil")
local i18n = require("Module:I18nUtil")
local lang = mw.getLanguage('en')

local p = require('Module:EntityAbstract'):extends()
local h = {}

p.objectType = 'Role'
p.imagelength = 'role'
p.imagesizes = {
	default = 15,
}
local MODIFIERS = {
	sub = {
		sortnumber = 20,
		default = 'Sub/',
		sentence = 'substitute ',
	},
	trainee = {
		sortnumber = 30,
		default = 'Trn/',
		sentence = 'trainee ',
	},
}
local MODIFIER_OVERRIDES = { sentence = true }

function p:init(str, opts)
	if not opts then opts = {} end
	self:super('init', str, 'Role')
	self.modifier = h.determineModifierFromInput(opts)
	self.defaultlength = 'name'
	if self.modifier and self.is_nil then
		self.is_nil = false
		self:super('init', self.modifier, 'Role')
		self.modifier = nil
	end
	if self.is_nil then return end
	if self.unknown then
		self.vars = {
			adjective = str,
			short = str,
			role = str,
			store = str,
			sentence = (str or ''):lower(),
			prep = 'for',
			article = 'a',
			sortnumber = 70,
			name = str,
			notaplayer = true,
		}
	end
	self.vars.notaplayer = util_args.nilToFalse(self.vars.notaplayer)
end

function p:name(opts)
	if not opts then opts = {} end
	if self.is_nil then return end
	if opts.skip_prefix then return self:super('name', opts) end
	if self:isStaff() then return self:super('name', opts) end
	if not self:hasPrefix() then return self:super('name', opts) end
	local len = opts.len or self.defaultlength
	local tbl = {
		self.modifier and MODIFIERS[self.modifier][len] or MODIFIERS[self.modifier].default or '',
		
		-- always return short when printing a modifier first unless we override
		-- (eg if sentence)
		self:get(MODIFIER_OVERRIDES[len] and len or 'short')
	}
	return util_table.concat(tbl, '')
end

function p:sortnumber(opts)
	if self.is_nil then return end
	local modSortNumber = self:hasPrefix() and MODIFIERS[self.modifier].sortnumber or 10
	return util_math.padleft(self:get('sortnumber') + modSortNumber, 2)
end

function p:sentence(opts)
	if self.is_nil then return nil end
	if not opts then opts = {} end
	local opts2 = mw.clone(opts)
	opts2.len = 'sentence'
	return ('%s %s'):format(self:name(opts2), self:get('prep'))
end

function h.determineModifierFromInput(opts)
	if opts.modifier then return opts.modifier:lower() end
	
	-- legacy support when we said sub = true, trainee = true
	if opts.Sub or opts.sub then return 'sub' end
	if opts.Trainee or opts.trainee then return 'trainee' end
	return nil
end

function p:hasPrefix()
	return self.modifier
end

function p:isIngame()
	return not self:get('notaplayer')
end

function p:isStaff()
	return self:get('notaplayer')
end

function p:image(opts)
	-- override for wikis that don't actually use roles
	if self.is_nil then return nil end
	if self.vars.name == 'Player' and not self.modifier then return nil end
	return self:super('image', opts)
end

function p:getImageClass(opts)
	if not self:isIngame() then return nil end
	if self.modifier == 'sub' then return 'sub' end
	if self.modifier == 'trainee' then return 'trainee' end
	return nil
end

function p:getSpriteKey(opts)
	if self:isStaff() and self.unknown then
		-- we'll overwrite the display so we can send any sprite we want for display
		return 'Staff'
	end
	local spriteKey = self:super('getSpriteKey', opts)
	return spriteKey
end

return p
```