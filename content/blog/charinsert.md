---
title: "CharInsert: Why you should \"char\""
date: 2020-05-05T04:46:37Z
draft: false
tags:
    - mediawiki
    - javascript
    - lua
summary: CharInsert is one of my favorite extensions. Here's how I made a Lua wrapper for it to create a user-friendly interface for anyone to add snippets they want.
description: CharInsert is one of my favorite extensions. Here's how I made a Lua wrapper for it to create a user-friendly interface for anyone to add snippets they want.
---

[CharInsert](https://www.mediawiki.org/wiki/Extension:CharInsert) is an extension that enables you to provide editors with links they can click to insert snippets into wikitext. It can be used for anything from providing one-click buttons to insert a single special character to serving as a complete alternative to [BoilerRoom](https://www.mediawiki.org/wiki/Extension:BoilerRoom) or [MultiBoilerplate](https://www.mediawiki.org/wiki/Extension:MultiBoilerplate), though I find it preferable even to either of these.

Before we continue, please note that "char" (as in "character") is pronounced the same as "care," and my title is a fantastic pun. Okay thanks.

{{% leaguepedia_citation %}}

## Base CharInsert functionality
CharInsert by default allows the user to click a piece of text to insert that piece of text. Spaces delineate separate pieces of text, but spaces can be escaped with `<nowiki>` tags.

Here is a simple example:
```
<charinsert><includeonly></includeonly><noinclude>{{documentation}}</noinclude></charinsert>
```
In fact, this is the first CharInsert I ever made. If you save this at `MediaWiki:Editnotice-10` (10 is the `Template` namespace) then any time you edit a template (using the source editor, not VE) you will see a piece of text styled like a hyperlink with the text `<includeonly></includeonly><noinclude>{{documentation}}</noinclude>` that you can click. Clicking it inserts the text into the textarea at the cursor.

We can slightly adjust this as follows, adding a `+` sign:
```
<charinsert><includeonly>+</includeonly><noinclude>{{documentation}}</noinclude></charinsert>
```
And now after clicking the insert, our cursor will be delivered in between the `<includeonly></includeonly>` at the `+`. Neat!

If we wanted to put multiple inserts in the same row next to each other, we could simply put spaces in between them, inside of the same extension tags. The canonical example on the documentation page is as follows:
```
<charinsert>Á á Ć ć É é Í í Ĺ ĺ Ń ń Ó ó Ŕ ŕ Ś ś Ú ú Ý ý Ź ź</charinsert>
```
In this case, we can click the "Á" to print `Á`, etc.

If, for some reason, we wanted to insert the literal text `Á á Ć ć É é Í í Ĺ ĺ Ń ń Ó ó Ŕ ŕ Ś ś Ú ú Ý ý Ź ź`, we could accomplish that as follows:
```
<charinsert>Á<nowiki> </nowiki>á<nowiki> </nowiki>Ć<nowiki> </nowiki>ć<nowiki> </nowiki>É<nowiki> </nowiki>é<nowiki> </nowiki>Í<nowiki> </nowiki>í<nowiki> </nowiki>Ĺ<nowiki> </nowiki>ĺ<nowiki> </nowiki>Ń<nowiki> </nowiki>ń<nowiki> </nowiki>Ó<nowiki> </nowiki>ó<nowiki> </nowiki>Ŕ<nowiki> </nowiki>ŕ<nowiki> </nowiki>Ś<nowiki> </nowiki>ś<nowiki> </nowiki>Ú<nowiki> </nowiki>ú<nowiki> </nowiki>Ý<nowiki> </nowiki>ý<nowiki> </nowiki>Ź<nowiki> </nowiki>ź</charinsert>
```
You can see then, that elaborate CharInserts could be rather unwieldy to define; we will certainly want a wrapper that provides some basic escaping. Here is the escaping I do in Lua:
```lua
function h.escape(insert)
    local replace = {
        ['\n'] = '&#10;',
        ['( +)'] = '<nowiki>%1</nowiki>',
        ['\\%['] = '[',
        ['\\%]'] = ']',
    }
    for k, v in pairs(replace) do
        insert = insert:gsub(k, v)
    end
    return insert
end
```
The first line escapes newlines, and the second escapes all sets of consecutive spaces. I've never needed to use a tab character in a CharInsert (at least not yet). The last two escapes undo the process of escaping brackets, since brackets are used as the delimiters for multi-line quotes in Lua, so I use the following to add a category: `\[\[Category:Fake Cargo Attach Templates\]\]` and then need to restore it to the desired `[[` and `]]` before printing.

So now, we've seen what CharInserts can do - but there's still a pretty big hole missing.

## Separate "label" attribute
What if we want to insert a giant wall of text? Say I want to let users input the following:
```
{{MatchSchedule|date= |time= |timezone= |dst= 
|<!-- Do not change the order of team1 and team2!! -->
|team1= |team2=
|team1score= |team2score= |winner=
|pbp= |color=
|vodinterview= |with=
|stream= |reddit=
|game1={{MatchSchedule/Game
 |blue= |red= |winner= |ssel= |ff=
 |mh=
 |recap=
 |vodpb=
 |vodstart=
 |vodpost=
 |vodhl=
 |vodinterview=
 |with=
 |mvp=
 }}
|game2={{MatchSchedule/Game
 |blue= |red= |winner= |ssel= |ff=
 |mh=
 |recap=
 |vodpb=
 |vodstart=
 |vodpost=
 |vodhl=
 |vodinterview=
 |with=
 |mvp=
 }}
}}
```

After escaping, it will look like this:
```
{{MatchSchedule|date=<nowiki> </nowiki>|time=<nowiki> </nowiki>|timezone=<nowiki> </nowiki>|dst=<nowiki> </nowiki>&#10;|<!--<nowiki> </nowiki>Do<nowiki> </nowiki>not<nowiki> </nowiki>change<nowiki> </nowiki>the<nowiki> </nowiki>order<nowiki> </nowiki>of<nowiki> </nowiki>team1<nowiki> </nowiki>and<nowiki> </nowiki>team2!!<nowiki> </nowiki>-->&#10;|team1=<nowiki> </nowiki>|team2=&#10;|team1score=<nowiki> </nowiki>|team2score=<nowiki> </nowiki>|winner=&#10;|pbp=<nowiki> </nowiki>|color=&#10;|vodinterview=<nowiki> </nowiki>|with=&#10;|stream=<nowiki> </nowiki>|reddit=&#10;|game1={{MatchSchedule/Game&#10;<nowiki> </nowiki>|blue=<nowiki> </nowiki>|red=<nowiki> </nowiki>|winner=<nowiki> </nowiki>|ssel=<nowiki> </nowiki>|ff=&#10;<nowiki> </nowiki>|mh=&#10;<nowiki> </nowiki>|recap=&#10;<nowiki> </nowiki>|vodpb=&#10;<nowiki> </nowiki>|vodstart=&#10;<nowiki> </nowiki>|vodpost=&#10;<nowiki> </nowiki>|vodhl=&#10;<nowiki> </nowiki>|vodinterview=&#10;<nowiki> </nowiki>|with=&#10;<nowiki> </nowiki>|mvp=&#10;<nowiki> </nowiki>}}&#10;|game2={{MatchSchedule/Game&#10;<nowiki> </nowiki>|blue=<nowiki> </nowiki>|red=<nowiki> </nowiki>|winner=<nowiki> </nowiki>|ssel=<nowiki> </nowiki>|ff=&#10;<nowiki> </nowiki>|mh=&#10;<nowiki> </nowiki>|recap=&#10;<nowiki> </nowiki>|vodpb=&#10;<nowiki> </nowiki>|vodstart=&#10;<nowiki> </nowiki>|vodpost=&#10;<nowiki> </nowiki>|vodhl=&#10;<nowiki> </nowiki>|vodinterview=&#10;<nowiki> </nowiki>|with=&#10;<nowiki> </nowiki>|mvp=&#10;<nowiki> </nowiki>}}&#10;}}
```
Uh....no thanks. I don't want my users *seeing* this, let alone having to know what happens when you click it. And it's actually even worse than you might think:
![Screenshot of the matchschedule CharInsert text](/images/charinsert/matchschedule-insert.png)

The text in `<pre>` is because I have some lines that start with a space, even though the space is escaped. You can still click it, and the desired text will input, but.....yeah, no.
    
So what we need is for a way to **decouple the display text from the inserted text**.

As it turns out, there's a [Phabricator thread](https://phabricator.wikimedia.org/T25354) discussing this that's been open for just over a decade as of my writing this post. So I gave up on the hope of having built-in support for this, and within a couple hours of finding the topic, I'd written my own support for this feature with just a couple lines of JS and one CSS rule (along with some conventions about how I'd print these things in HTML):
```js
$(function() {
    $('.mw-charinsert-item').each(function() {
        $(this).html($(this).closest('div').attr('data-ci-label'));
        $(this).css('display', 'inline-block');
    });
    $('.ci-loading-text').css('display','none');
});
```

```css
.client-js .mw-charinsert-item {
 display:none;
}
```

Here's what this does:

* The user creates a CharInsert using the `<charinsert>` tag. This results in a container with class `.mw-charinsert-item`. The text given to the CharInsert should be the desired *insertion* text.
* The Lua wrapper wraps each CharInsert in a `div` which is given an attribute `data-ci-label` with the desired *display* text.
* The Lua wrapper further adds a line that says `Loading...` or something similar, to prevent vertical jumps of content as the page loads.
* Prior to this point, the css rule `.client-js .mw-charinsert-item { display:none; }` is in effect; this ensures that the user will not see a scary jumble of 100 lines of escaped wikitext.
* Now that the substitution has been made, the `display` on `.mw-charinsert-item` is reverted to `inline-block`.
* Finally, we hide the loading text.

Of course, this all assumes a bunch of specifics about my HTML, but since I'm going to be generating everything in Lua, that's not an issue at all.

## Deploying CharInserts
Before I show you my Lua code, let's talk briefly about how to deploy CharInserts for use.

### Interlude: Finding system messages
If you add the argument `uselang=qqx` to the URL of any page (if there's already one or more specified params, put `&uselang=qqx`; if not, then `?uselang=qqx`) will show you the names of system messages instead of their contents.

So, say I edit my user page on Leaguepedia. Normally the url is `https://lol.gamepedia.com/index.php?title=User:RheingoldRiver&action=edit`, but I want to see what system messages are present, so I'll instead go to `https://lol.gamepedia.com/index.php?title=User:RheingoldRiver&action=edit&uselang=qqx`. (Note that the `?` comes earlier in the URL, before the first parameter, which is `title=User:RheingoldRiver`.)

### Back to deploying CharInserts
So on my user page, I'm shown the following four system message names:
```
(editnotice-2)

(editnotice-2-RheingoldRiver)

(longpage-hint: (size-kilobytes), 1115)

(editpage-head-copy-warn)
```
The last one is included on every page on the entire wiki, but I'd rather avoid using the copyright warning to display editor tools. Instead I'm going to use the first message, which is `editnotice-2`. If I get a list of all of my namespaces (doable either by going manually through `Special:AllPages` or using the API), I can get a list of system messages I will have to display:
* `MediaWiki:Editnotice-0` (main)
* `MediaWiki:Editnotice-1` (talk)
* `MediaWiki:Editnotice-2` (user)
* `MediaWiki:Editnotice-3` (user talk)
* etc.

I'm going to place the same text on all of them, except for the first:
```
{{int:Editnotice-0}}
```
This internationalizes (translates) the edit notice for the main namespace and displays it on each other namespace's edit notice.

At `MediaWiki:Editnotice-0` I have the following:
```
Thanks for editing the wiki! Want to join our Discord server? {{DiscordURL}}
{{#invoke:CharInserts|main}}
```
The first line is unrelated to CharInsert; the second simply invokes my CharInserts module.

As it happens, I actually want my CharInserts to behave differently depending on namespace - I have some preloads that are template-only, some that are main-only, etc. However, I chose to do all of this handling inside of Lua, so in the system messages I'm merely invoking the same module on all pages.

## Config files
See also [this post](https://river.me/blog/cargo-docstrings/) about how to design nice config files.

As the last thing before we move to the actual Lua code, let's look how I've arranged my config files. The goal is to have something that editors can update without knowing any Lua or programming at all.

As I mentioned, I have [different behaviors depending on namespace](https://lol.gamepedia.com/Category:CharInsert_Modules). I also have different behaviors depending on pattern matches against the title of the current page. In particular, the following parameters are provided for each insert:

* `pattern`, a required pattern to appear somewhere in the title; if this is the empty string, then every title matches it by default;
* `notpattern`, a pattern or list of patterns that must NOT appear in the title if the insert is to appear; this takes precedence over `pattern`;
* `label`, the human-readable text to display for an editor to click; and
* `insert`, the actual text (human-readable and unescaped) that the CharInsert will add to the page.

Because often `notpattern` is a list of "pages with patterns that are handled specially," at the start of most config files I define a constant called `NOT_PATTERNS` that will be used as the `notpattern` for many inserts. (This is also why config files as code are better than standalone non-code text files.)

Here is an excerpt from my `Main` namespace CharInserts. Note that indentation style is not preserved mid-insert to avoid sending careless whitespace to the charinsert. At the same time, I deliberately arranged `insert` to be the final item in each group, so that I didn't need to change indentation after it.

```lua
local NOT_PATTERNS = {
    "/Team Rosters",
    "/Casting History",
}

return {
    {
        pattern = '/Casting History',
        label = "Insert Casting History",
        insert = [[{{PlayerTabsHeader}}
{{CastingHistory}}]]
    },
    {
        pattern = "",
        notpattern = NOT_PATTERNS,
        label = "Insert skin page",
        insert = [[{{Infobox Skin
|champion=
|name=
|rp=
|date=
|legacy=
|artists=
|artistlinks=
}}
{{ChampionSkinIntro}}
{{ChampionSkinImageSections}}
]]
    },
    {
        pattern = ' Teams$',
        label = "Insert TeamCountryPortal",
        insert = "{{TeamCountryPortal|+}}"
    },
    {
        pattern = ' Teams$',
        label = "Insert TeamRegionPortal",
        insert = "{{TeamRegionPortal|+}}"
    },
    {
        pattern = '/Scoreboards',
        label = "Insert Scoreboard/Tab",
        insert = [[{{Scoreboard/Tab|+|continue=}}]]
    },
}
```

### Pagename
I want to point one final thing out about the peculiarities of escaping for CharInsert: Normally, if I want the title of a page inserted into that page, I can simply used `{{subst:PAGENAME}}` or `{{subst:BASEPAGENAME}}` or what have you. However, `subst` does NOT work inside of `<includeonly>` tags.

Therefore, for one of my inserts for the `Template` namespace, when I want to invoke a Lua module with the same title as the template, I write the following:
```
<includeonly>{{#invoke:$PAGENAME$|main}}</includeonly><noinclude>{{documentation}}</noinclude>
```
the string `$PAGENAME$` follows other conventions for formatting parameters into longer strings that I use elsewhere on the wiki, so part of my escaping/formatting is the following:
```lua
function h.makeReplacements(str)
    local replacements = {
        PAGENAME = mw.title.getCurrentTitle().text,
    }
    return util_sentence.makeReplacements(str, replacements)
end
```

And the code in `Module:SentenceUtil` for `makeReplacements` is as follows:
```lua

function p.makeReplacements(sentence, replacements)
    if not sentence then return nil end
    if not replacements then replacements = {} end
    local old = sentence
    for k, v in pairs(replacements) do
        sentence = sentence:gsub('%$' .. k .. '%$', v or 'unknown')
    end
    if sentence == old then
        return sentence
    end
    return p.makeReplacements(sentence, replacements)
end
```
(Note the recursive capacity of `replacements` - while it's not needed here, in other situations, it's required.)

In the future, if I find other necessary pre-printing substitutions to make, I can simply append additional keys to the table `replacements` in the first snippet above.

## The Lua wrapper!
Finally, we need to turn our config file into HTML/wikitext laid out the way our JS expects. Here is the full code of the wrapper. Note the inclusion of the previous escaping/replacement snippets:
```lua
local util_sentence = require("Module:SentenceUtil")
local util_table = require('Module:TableUtil')
local util_vars = require("Module:VarsUtil")
local i18n = require('Module:i18nUtil')

local h = {}

local p = {}
function p.main(frame)
    local title = mw.title.getCurrentTitle()
    i18n.init('CharInserts')
    return h.makeOutput(h.getData(title.nsText), title.text)
end

function h.getData(nsText)
    local moduleTitle = mw.title.new(h.getModuleName(nsText), 'Module')
    if not moduleTitle.exists then return nil end
    return mw.loadData(moduleTitle.prefixedText)
end

function h.getModuleName(nsText)
    if nsText == '' then nsText = 'Main' end
    return ('CharInserts/%s'):format(nsText)
end

function h.makeOutput(data, title)
    if not data then return end
    local ret = {}
    for _, group in ipairs(data) do
        ret[#ret+1] = h.getGroupOutput(group, title)
    end
    return unpack(ret)
end

function h.getGroupOutput(group, title)
    if not h.containsOneOf(title, group.pattern) then return nil end
    if h.containsOneOf(title, group.notpattern or '@@@@@@@@@@') then return nil end
    local div = h.makeDiv(group.label)
    h.printContent(div, group)
    return div
end

function h.containsOneOf(title, patterns)
    patterns = util_table.guaranteeTable(patterns)
    for _, v in ipairs(patterns) do
        if title:find(v) then
            return true
        end
    end
end

function h.makeDiv(label)
    return mw.html.create('div'):attr('data-ci-label', label)
end

function h.printContent(div, group)
    div:wikitext(h.loadingText(group.label), h.makeContent(group.insert))
end

function h.loadingText(label)
    local output = mw.html.create('span')
        :addClass('ci-loading-text')
        :wikitext(i18n.print('loading', label or ''))
    return tostring(output)
end

function h.makeContent(insert)
    return mw.getCurrentFrame():extensionTag{
        name = 'charinsert',
        content = h.makeReplacements(h.escape(insert)),
        args = {}
    }
end

function h.makeReplacements(str)
    local replacements = {
        PAGENAME = mw.title.getCurrentTitle().text,
    }
    return util_sentence.makeReplacements(str, replacements)
end

function h.escape(insert)
    local replace = {
        ['\n'] = '&#10;',
        ['( +)'] = '<nowiki>%1</nowiki>',
        ['\\%['] = '[',
        ['\\%]'] = ']',
    }
    for k, v in pairs(replace) do
        insert = insert:gsub(k, v)
    end
    return insert
end

return p
```