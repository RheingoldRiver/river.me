---
title: "Lua Hooks System"
date: 2020-03-09T10:44:27Z
draft: false
tags:
    - mediawiki
    - lua
summary: A system of hooks in Scribunto can be reasonable to use if you have a wikifarm with synced code that occasionally needs to be slightly modified on one or more wikis. I found the system more complicated than it was worth, but I'm documenting it here before I remove it.
---

## Introduction
I'm deleting my hooks system.

I'm deleting my hooks system because I don't think I need it (at least not currently). The reasons I built it at first include:
* I didn't have a robust class system yet (now I use [LuaClassSystem](https://github.com/Yonaba/Lua-Class-System))
* I wanted to be able to blindly copy-paste pages of code from one wiki to another, but still retain the ability to have wiki-specific functionality by storing certain behavior on standalone unsynced pages.
* I wanted to allow other people to modify my code in certain special cases by just writing short functions.

The first reason isn't really a valid reason for using hooks; LuaClassSystem works sufficiently well, and I'd recommend its use (though we made some changes to it on Leaguepedia, none of which I fully understand, but all of which address some problematic behavior I was experiencing). An introduction to this code is a planned future post for this blog.

But the last two reasons are actually legitimate reasons for using hooks. Currently I've decided it's not worth it, and as of writing this post I've deleted all usage of hooks from my code except for the one I'm documenting here; after this post is published I'll delete it as well. That said, if you are in a position where you want to make certain things easily customizable on one wiki out of a group that share Lua code (perhaps by scary transclusion, perhaps by automated processes to sync code), it might be reasonable to consider hooks.

{{% leaguepedia_citation %}}

## Building a hook system

The code to execute hooks was contained in `Module:Hook`:
```lua
local p = {}

function p.add(k, v)
    if not INTERNAL_HOOK_TABLE then
        -- set global table of hooks
        INTERNAL_HOOK_TABLE = {}
    end
    if not INTERNAL_HOOK_TABLE[k] then
        INTERNAL_HOOK_TABLE[k] = {}
    end
    INTERNAL_HOOK_TABLE[k][#INTERNAL_HOOK_TABLE[k]+1] = v
end

function p.run(key, ...)
    if INTERNAL_HOOK_TABLE and INTERNAL_HOOK_TABLE[key] then
        for _, f in ipairs(INTERNAL_HOOK_TABLE[key]) do
            if not f(...) then
                return false
            end
        end
    end
    return true
end

return p
```

## Using hooks
Using a hook looked something like this:

`Module:MVPHistory` (adding hooks to `Module:MatchHistoryPlayer`):
```lua
function p.main(frame)
    local args = util_args.merge(true)
    Hook.add('onMatchHistoryPlayerGetTables', h.onMatchHistoryPlayerGetTables)
    Hook.add('onMatchHistoryPlayerGetJoin', h.onMatchHistoryPlayerGetJoin)
    h.setOnMatchHistoryPlayerGetWhere(args.player, args.mvptype)
    h.setOnMatchHistoryPlayerPrintColspanHeader(args.player, args.mvptype)
    args.preload='Player'
    args.limit=args.limit or 50
    args.link=args.player
    args.nostats = 'yes'
    return MatchHistoryPlayer(args)
end

function h.onMatchHistoryPlayerGetTables(tbl)
    tbl[#tbl+1] = 'MatchScheduleGame=MSG'
    tbl[#tbl+1] = 'MatchSchedule=MS'
    return true
end

-- snip

function h.setOnMatchHistoryPlayerPrintColspanHeader(player, mvptype)
    local function onMatchHistoryPlayerPrintColspanHeader(tbl, colspan)
            util_html.printColspanHeader(tbl, i18n.print('mvp_' .. mvptype:lower(), player), colspan)
    end
    Hook.add('onMatchHistoryPlayerPrintColspanHeader', onMatchHistoryPlayerPrintColspanHeader)
end
```

And then they were run from the parent module, in this case `Module:MatchHistoryPlayer`:
```lua
function h.getTables(args)
    local ret = {
        'ScoreboardGame=SG',
        'Tournaments=IT',
        'ScoreboardPlayer=SP',
        'ScoreboardPlayer=SPVs',
        'PlayerRedirects=PR',
    }
    Hook.run('onMatchHistoryPlayerGetTables', ret)
    return ret
end

-- snip

function h.printColspanHeader(tbl, args)
    if PRELOAD.noheading then return end
    if not Hook.run('onMatchHistoryPlayerPrintColspanHeader', tbl, #PRELOAD.headings) then return end
    local displayTbl = {
        util_stats.heading(args, 'MatchHistoryPlayer', h.getLimit(args)),
        util_stats.openAsQueryLink(SETTINGS.form_info, args)
    }
    util_html.printColspanHeader(tbl, util_table.concat(displayTbl, ' - '), #PRELOAD.headings)
end
```
If no hook function is defined, `Hook.run` would return `true`, and execution in `h.printColspanHeader` continues. If a hook function is defined, and returns `true`, then the `Hook.run` also returns `true`, and execution continues. But if the hook function returns `false`, then execution of further hooks is interrupted immediately after that function concludes, and in `h.printColspanHeader` we also halt further execution.

## Documenting hooks
Because they kinda just throw code around, I wanted to be absolutely certain that I documented hooks extremely thoroughly, which is equivalent to being absolutely certain I documented them *automatically*.

I also had a very strict naming convention. A hook was called `on[ModuleName][FunctionName]`. A single function in a module couldn't have more than one hook (if this became a pain point, probably I would have allowed some convention for that as well).

The following is displayed on modules that run hooks:

![Screenshot of HooksRun documentation](/images/lua-hooks-system/hooks-run.png)
> This module adds the following hooks:
> 
> * onMatchHistoryPlayerPrintColspanHeader
> * onMatchHistoryPlayerGetJoin
> * onMatchHistoryPlayerGetWhere
> * onMatchHistoryPlayerGetTables

And the following is displayed on modules that define (add) hooks:

![Screenshot of HooksAdded documentation](/images/lua-hooks-system/hooks-added.png)
> This module runs and makes the following hooks available:

> * onMatchHistoryPlayerGetWhere - added by: `{{MVPHistory}}`
> * onMatchHistoryPlayerGetJoin - added by: `{{MVPHistory}}`
> * onMatchHistoryPlayerPrintColspanHeader - added by: `{{MVPHistory}}`
> * onMatchHistoryPlayerGetTables - added by: `{{MVPHistory}}`


### Cargo
Once raw data is added to the wiki, I defined the following table so that I could query all potential sources and destinations of hooks as desired:
```lua
return {
    { field = "Hook", type = "String", desc = "Hook added or run" },
    { field = "Module", type = "Page", desc = "Module name (without doc subpage)" },
    { field = "Action", type = "String", desc = "Added or Run" },
}
```

For the example modules, this resulted in the following data:

| Page | Hook | Module | Action
|-------------------------------    |----------------------------------------   |---------------------------    |-------    |
| Module:MatchHistoryPlayer/doc     | onMatchHistoryPlayerGetJoin               | Module:MatchHistoryPlayer     | Run       |
| Module:MatchHistoryPlayer/doc     | onMatchHistoryPlayerGetTables             | Module:MatchHistoryPlayer     | Run       |
| Module:MatchHistoryPlayer/doc     | onMatchHistoryPlayerGetWhere              | Module:MatchHistoryPlayer     | Run       |
| Module:MatchHistoryPlayer/doc     | onMatchHistoryPlayerPrintColspanHeader    | Module:MatchHistoryPlayer     | Run       |
| Module:MVPHistory/doc             | onMatchHistoryPlayerGetJoin               | Module:MVPHistory             | Added     |
| Module:MVPHistory/doc             | onMatchHistoryPlayerGetTables             | Module:MVPHistory             | Added     |
| Module:MVPHistory/doc             | onMatchHistoryPlayerGetWhere              | Module:MVPHistory             | Added     |
| Module:MVPHistory/doc             | onMatchHistoryPlayerPrintColspanHeader    | Module:MVPHistory             | Added     |

### Python
In order to maintain updated docs, I had the following Python code:
```python
import re, mwparserfromhell
from river_mwclient.esports_client import EsportsClient
from river_mwclient.auth_credentials import AuthCredentials

interval = 180

pattern_add = r'Hook.add\([\'"](\w+).*\)'
pattern_run = r'Hook.run\([\'"](\w+).*\)'

credentials = AuthCredentials(user_file="me")
site = EsportsClient('lol', credentials=credentials) # Set wiki

revisions = site.client.recentchanges_by_interval(interval, toponly=1)

def add_missing_params(template, params_to_add):
    n = 0
    for param in template.params:
        n += 1
        param_str = str(param.value)
        if param_str in params_to_add:
            params_to_add.remove(param_str)
    for param in params_to_add:
        template.add(n + 1, param)
    params_to_add.clear()
    

def add_new_template(text, template_name, params):
    if not len(params):
        return text
    template = mwparserfromhell.nodes.Template(template_name)
    n = 1
    for param in params:
        template.add(n, param)
        n += 1
    if text == '':
        return str(template)
    return text + '\n' + str(template)

for revision in revisions:
    title = revision['title'].replace('/doc', '')
    if not title.startswith('Module:'):
        continue
    module = site.client.pages[title]
    doc = site.client.pages[title + '/doc']
    text = module.text()
    added = set()
    run = set()
    for match in re.findall(pattern_add, text):
        added.add(match)
    for match in re.findall(pattern_run, text):
        run.add(match)
    doc_text = doc.text()
    wikitext = mwparserfromhell.parse(doc_text)
    for template in wikitext.filter_templates():
        if template.name.matches('HooksAdded'):
            add_missing_params(template, added)
        elif template.name.matches('HooksRun'):
            add_missing_params(template, run)
    new_doc_text = str(wikitext)
    new_doc_text = add_new_template(new_doc_text, 'HooksAdded', added)
    new_doc_text = add_new_template(new_doc_text, 'HooksRun', run)
    if new_doc_text != '' and (doc_text == '' or not doc_text):
        new_doc_text = '<includeonly>{{luadoc}}[[Category:Lua Modules]]</includeonly>\n' + new_doc_text
    if new_doc_text and doc_text != new_doc_text and new_doc_text.strip() != '':
        doc.save(new_doc_text, summary="Auto updating hook documentation page")
```
This code scans recent revisions, looks for added and/or run hooks in any Lua code it finds, and updates the docs to either add or remove any hooks it discovers. My `mwclient` wrapper is [here](https://github.com/RheingoldRiver/river_mwclient).

### Lua
```lua
local util_args = require('Module:ArgsUtil')
local util_cargo = require('Module:CargoUtil')
local util_table = require('Module:TableUtil')
local util_text = require('Module:TextUtil')
local i18n = require('Module:i18nUtil')

local h = {}

local p = {}
function p.added(frame)
    local args = util_args.merge(true)
    if mw.title.getCurrentTitle().text:find('/doc$') then
        h.storeCargo(args, 'Added')
    end
    i18n.init('HookDoc')
    return h.makeOutput(args, 'Added')
end

function p.run(frame)
    local args = util_args.merge(true)
    if mw.title.getCurrentTitle().text:find('/doc$') then
        -- can't store cargo from lua content model page
        h.storeCargo(args, 'Run')
    end
    i18n.init('HookDoc')
    return h.makeOutput(args, 'Run')
end

function h.storeCargo(args, hooktype)
    for _, hook in ipairs(args) do
        util_cargo.store({
            _table = 'Hooks',
            Hook = hook,
            Action = hooktype,
            Module = mw.title.getCurrentTitle().rootPageTitle.prefixedText,
        })
    end
end

function h.makeOutput(args, doctype)
    local output = mw.html.create()
    output:wikitext(i18n.print('intro' .. doctype))
    local ul = output:tag('ul')
    for _, hook in ipairs(args) do
        ul:tag('li'):wikitext(h.makeDisplayText(hook, doctype))
    end
    return output
end

function h.makeDisplayText(hook, doctype)
    if doctype == 'Added' then
        return util_text.intLink(h.getRunPageForAdded(hook), hook)
    elseif doctype == 'Run' then
        return hook, ' ', i18n.print('addedBy'), h.getAddedListForRun(hook)
    end
end

function h.getRunPageForAdded(hook)
    return util_cargo.getOneResult({
        tables = 'Hooks',
        fields = 'Module',
        where = ('Action="Run" AND Hook="%s"'):format(hook)
    })
end

function h.getAddedListForRun(hook)
    return util_table.concat(util_cargo.getOrderedList({
        tables = 'Hooks',
        fields = 'Module',
        where = ('Action="Added" AND Hook = "%s"'):format(hook)
    }), ', ', h.moduleDisplay)
    
end

function h.moduleDisplay(str)
    return mw.getCurrentFrame():expandTemplate{
        title = 'mod',
        args = { str:gsub('Module:', ''), '' },
    }
end

return p
```

The main thing to notice here is the following:
```lua
if mw.title.getCurrentTitle().text:find('/doc$') then
    h.storeCargo(args, 'Added')
end
```
Here I actually have to check for the page name ending in `/doc` to store, because Cargo doesn't store from module pages. This requires the documentation page to look like this:

```
<includeonly>{{luadoc}}[[Category:Lua Modules]]</includeonly>
{{HooksRun|onMatchHistoryPlayerGetWhere|onMatchHistoryPlayerGetJoin|onMatchHistoryPlayerPrintColspanHeader|onMatchHistoryPlayerGetTables}}
```

With the `HooksRun` code very deliberately outside of the `includeonly`.

## Conclusion
All things considered, this is actually not that much code to build out a framework, especially when you consider that the majority of code is for documentation purposes. That said, it feels a bit hacky to do, and I wouldn't necessarily recommend its use ever. But it can be a way to allow code to be modified locally while documenting changes as thoroughly as possible.