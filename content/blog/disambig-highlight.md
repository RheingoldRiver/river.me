---
title: "Highlighting disambiguations"
date: 2020-05-19T13:21:48Z
draft: false
tags:
    - mediawiki
    - javascript
    - lua
    - cargo
    - css
summary: Disambiguations are typically somewhat user-unfriendly, especially when a wiki has a relatively low percentage of pages that are disambiguated and so the presence of one is unexpected. Here I describe my method for improving quality of life when disambiguation links do appear.
---
Today on Leaguepedia, if you encounter a disambiguation link, it will be highlighted in bold orange, so you know something is wrong. Additionally, there is a button next to it that you can click to see a pop-up with all of the possible player disambiguation links, which you can choose to follow or copy. And hovering each one shows a further tooltip with some information about the player. (Disambiguations of other types aren't yet supported, but that's on my todo list. It's a pretty nontrivial task for reasons I'll explain later.)

A table showing a link that should have been disambiguated, but wasn't:
![Screenshot of highlighted disambiguation](/images/disambig-highlight/highlighted-disambig.png)

And here is the pop-up displaying the options that the editor might want to use:
![Screenshot of highlighted disambiguation](/images/disambig-highlight/highlighted-popup.png)

On player pages, a disambiguation link is printed deliberately when applicable; here, the link isn't highlighted, but the button still exists:
![Screenshot of current disambiguation tooltip](/images/disambig-highlight/current-popup.png)

Crucially, this entire setup requires no additional effort or special knowledge on the part of an editor. Everything is done using CSS and JavaScript that automatically loads whenever a disambiguation link is printed on the wiki, anywhere, even if it just uses the normal `[[Link]]` syntax.

Throughout this post I've included numerous code snippets; some of them are modified to omit details not yet relevant to the discussion, and some of them appear multiple times. Therefore I've included a full list of all code files at the end of this post, which you can read or copy or ignore. [Click here](#code-files) to skip to the code (or to check the scroll bar to see how much of this you don't have to read).

{{% leaguepedia_citation %}}

## Highlighting disambigs
The first thing I did was simply to highlight links to disambiguations in a very noticeable style, so that there was a clear signal to the editor that something was wrong.
### Initial method
There's not really any nice way to put this, I brute-forced the entire thing:
```css
a[title="Kìrìt0"], a[title="2015 League Championship Series"], a[title="AK"], a[title="AKi"], a[title="Absolute"], a[title="Ace"], a[title="Aegis"], a[title="Aesthetic"], a[title="Aki"], a[title="Akira"], a[title="Aliez"], /* 

snip

*/ a[title="Von"], a[title="Vulcan"], a[title="WaY"], a[title="Ward"], a[title="Way"], a[title="Wei"], a[title="Willy"], a[title="Wind"], a[title="Winnie"], a[title="Wolf"], a[title="Wolves"], a[title="Woong"], a[title="World GameMaster Tournament"], a[title="XiaoHan"], a[title="Xiaohan"], a[title="Xuan"], a[title="Xx"], a[title="Yan"], a[title="Yang"], a[title="YoDa"], a[title="Yoshi"], a[title="Yuuki"], a[title="ZZZ"], a[title="Zen"], a[title="Zero"], a[title="Zest"], a[title="Ziv"], a[title="Zzz"] {
    color:orange!important;
    font-weight:bold;
}
```
.....yeah. At the time I replaced this method, there were 388 links included. It was not pleasant. That said, most of the code was generated, not manually curated:

```python
css_style = " {\n    color:orange!important;\n    font-weight:bold;\n}"

def save_css_page():
    print('Starting css page...')
    csspage = site.pages["MediaWiki:Gadget-highlightDisambigs.css"]
    csstext = csspage.text()
    if '"' + original_name + '"' not in csstext:
        # use re in case a human edited the page and didn't use exactly the expected styling
        # remove the style from the string
        s = csstext.split('{')[0]
        # split string to capture the page titles
        tbl = re.split('a\\[title="\s*(.+?)\s*\"\\],?\s*', s)
        tbl.append(original_name)
        tblSorted = sorted(tbl)
        # re-add style
        tblSorted2 = ['a[title="{}"]'.format(s) for s in tblSorted if s.strip(", ") != ""]
        # concatenage back into a string
        csstext = ', '.join(tblSorted2) + css_style
        print("Saving css page...")
        csspage.save(csstext, summary=summary, tags="bot_disambig")
```
This code was part of a much larger script - in fact, the first Python script I ever wrote for the wiki - that scanned through all backlinks of a player page needing to be disambiguated and updated them to use the new name. Along the way it moved the page to its disambiguated location, did some blank edits, and updated this css file. It also alphabetized the list in the file, so if people ever added links by hand, out of order, they'd get alphabetized properly the next time I disambiguated a page using the script.

### Highlight links in category
In mid 2019, I discovered the extension [Highlight Links in Category](https://www.mediawiki.org/wiki/Extension:Highlight_Links_in_Category). My immediate thought was, "I should replace my disambigs with this!" So I did.

Here's my current value for `$wgHighlightLinksInCategory`:
```php
{
  "Disambiguations": "catlink-disambiguations",
  "Players": "catlink-players",
  "Teams": "catlink-teams",
  "Tournaments": "catlink-tournaments",
  "TeamsWithAutomatedNews": "tWAN"
}
```
There was some issue I encountered when categories here had spaces in their names, so for the last one, which is a temporary maintenance category, I just removed all spaces from the category name. It also has an abbreviated class name because it's a temporary thing so I figured may as well make it short.

The relevant item here is the first one - all links to disambiguations will have the class `catlink-disambiguations` added to them.

My entire css snippet above can now be replaced with the following:
```css
.catlink-disambiguations {
    color:orange!important;
    font-weight:bold;
}
```
Nice!
#### Don't highlight intentional disambig links
There's actually one other rule here, which handles the case you saw in the introduction of a deliberate link to a disambig. So the full contents of my `highlightDisambigs.css` gadget is as follows:
```css
.catlink-disambiguations {
    color:orange!important;
    font-weight:bold;
}

.sbs-disambig .catlink-disambiguations {
 color:var(--link)!important; /* im so sorry */
 font-weight:normal;
}
```
## Disambiguation tooltip buttons
So, highlighting disambiguations is great for discovery that there's a problematic link. But the process for correcting this link still sucks - one must open the page to the disambiguation, copy the right name, close tab/navigate back to the original page, and paste it in. We'd like to get rid of the navigation step altogether.
### The on-load JavaScript way
So the easy way to do this would be to add a node (probably a `span`) after each `a.catlink-disambugations` with JS, which, when clicked, would show a pop-up with all the disambiguations. But screw that, I want a CSS-on-load-only method that doesn't involve any JS until the user actually clicks something (other than assigning an onclick function).
### The CSS way
So...let's make an `::after` pseudo-element! Not that hard:
```css
.catlink-disambiguations {
    position:relative;
}

.catlink-disambiguations::after {
    font-family:"FontAwesome";
    content:"\f05a";
    cursor:pointer;
    color:var(--body-text-color);
}
```
The thing is, when you click this `::after`, your click event is sent to the parent link, which just navigates you. Not what we want. We're gonna have to figure out where exactly in this link the click happened and then apply an onclick event conditionally.

First let's adjust the CSS so this is possible to do:
```css
.catlink-disambiguations {
    position:relative;
    margin-right:1em;
}

.catlink-disambiguations:not(.catlink-disambiguations-activated)::after {
    font-family:"FontAwesome";
    content:"\f05a";
    cursor:pointer;
    color:var(--body-text-color);
    position:absolute;
    right:-1em;
}
```
Our situation is now that the full width of our new element is exactly the same as it was before; however, instead of the space allotted to the `::after` being part of the *main* element, it's positioned "on top of" the element's right margin. This lets us access the x-coordinate of the boundary of the `::after` using the `innerWidth()` function, as follows:
```js
    var el = this;
    if (e.pageX <= $(el).offset().left + $(el).innerWidth()) return;
    e.preventDefault();
```
Now that we have the "tech" to show the button on page load, without any visible-to-the-user JavaScript, let's switch gears and talk about the actual *content* of this thing. Remember part of our premise was that the entire system must work without the use of any templates or special markup. So we need a method for the contents of the pop-up to be created based on no information other than the normal attributes of the link - i.e. the title/href.

## Disambiguation tooltip contents
I'll preface this with a note that I intend to revise this structure, and when I do, I'll write a followup post explaining the limitations of this method and my solution. But this is the code I'm currently running as of writing this post, and it works sufficiently well that it's already a great feature.
### Storing data
If you've read any of my previous posts, it should be pretty obvious I'm going to do this with [Cargo](https://river.me/tags/cargo/). Here's the definition of my Disambiguations table (recall I [wrap declarations in Lua](https://river.me/blog/cargo-docstrings/)):

```lua
return {
    { field = 'FinalLocation', type = 'String', desc = 'The page where the term is located' },
    { field = 'Term', type = 'String', desc = 'Will be _pageName unless the disambig is located at _(Disambiguation)' },
    { field = 'DisambigType', type = 'String', desc = 'Player, Team, Other' },
    { field = 'N_LineInPage', type = 'String', desc = '' },
    { field = 'DisambigID', type = 'String', desc = 'To join to type-specific tables' },
}

```
The `Term` field solves cases like [Uzi (Jian Zi-Hao)](https://lol.gamepedia.com/Uzi_\(Jian_Zi-Hao\)), where the disambiguation page is located at [Uzi (Disambiguation)](https://lol.gamepedia.com/Uzi_\(Disambiguation\)), and [Uzi](https://lol.gamepedia.com/Uzi) redirects to `Uzi (Jian Zi-Hao)`. In this case I want to query for the term being `Uzi`, but `_pageName` will be `Uzi (Disambiguation)`. The field `N_LineInPage` lets me duplicate the order in which disambiguations are listed from the page itself to the tooltip or other display. In theory `FinalLocation` and `DisambigID` should be the same, but I do not like having page names double as joining keys.

The module that stores to this table is called `DisambigPage`, and it's stored from the disambiguation pages themselves. I'll provide a snippet of Lua code; note that I'm extremely lazy here and make one database call per item in the disambiguation list; however, this is not really a problem because there's almost never more than 3 or 4 items on a single page, and the absolute maximum is about 10.

```lua
function h.getAndStorePlayerData(args)
    local tbl = util_args.numberedArgsToTable(args, 'player')
    if not tbl then return nil end
    util_map.arrayInPlaceWithIndex(tbl, h.getPlayerData, args)
    h.storeCargo(tbl, 'Player')
    return util_map.inPlace(tbl, h.makePlayerLine)
end

function h.getPlayerData(player, i, args)
    if mw.title.makeTitle('', player).exists then
        return h.getPlayerDataFromCargo(player)
    end
    return h.getPlayerDataFromArgs(args, i)
end

function h.getPlayerDataFromCargo(player)
    local row = util_cargo.getOneRow(h.makePlayerQuery(player))
    row.IsFormer = not row.Team
    row.Team = row.Team or row.TeamLast
    row.CurrentName = row._pageName ~= lang:ucfirst(player) and row._pageName
    row.Name = player
    return row
end

function h.makePlayerQuery(player)
    return {
        tables = 'Players, PlayerRedirects=PR',
        join = 'Players._pageName=PR._pageName',
        fields = h.fields,
        groupBy = 'Players._pageName',
        where = ('PR.AllName = "%s"'):format(player),
        types = {
            IsSubstitute = 'boolean',
            IsLowercase = 'boolean'
        }
    }
end
```
This snippet is missing some details, but they aren't particularly important. Of course, there's similar functions for storing teams. At the end of this post I've included the complete source of all mentioned modules and gadgets.
#### Limitations of this approach
Briefly, I dislike this entire approach because it requires a disambiguation to know too much:
* What types of entities are there?
* How do you construct a display sentence for each type of entity?
* What fields are relevant to query for each entity?

Really, the disambiguations should be set on the target pages themselves and stored in a generic `Entities` table with a field called `DisambigSentence` that can be queried as needed. This new method would also allow for a single query to discover all disambiguations and display them without any knowledge of types of pages or secondary details. Then disambiguation pages only need to know:
* What pages have this ID?
* What do they want their sentences to be?

This is a much more reasonable list than that of the original approach.

I've glossed over one issue in the above (which is actually a big enough issue that it's the reason I haven't changed the setup over yet) - currently, we have freedom to do things like show a player nicknamed `Claps` on the disambiguation page `Clap` - and this is not so easily automated. So a method for manually adding players or pages with similar but not identical names to a disambiguation is needed, and this rather complicates the final product.

Also remember when I mentioned the order parameter in the Cargo query? Yeah, if I switch to storing everything on individual pages, and the disambiguation pages query to discover what to list, I'd have to do alphabetical order or something, instead of allowing editors to set their own ordering.
### Querying data
So, anyway, even in the current setup we have an easy way to discover all disambiguations associated to a particular term - just query `Disambiguations`. By stripping all characters including and after the first `(` in a title attribute, we can find the value of `Disambiguations.Term` and from there get a list of all entities with the same term.

Here are the contents of `Module:DisambigPage`:

```lua
local util_args = require('Module:ArgsUtil')
local util_cargo = require("Module:CargoUtil")
local util_html = require("Module:HtmlUtil")
local util_table = require("Module:TableUtil")
local util_text = require("Module:TextUtil")
local util_toggle = require("Module:ToggleUtil")

local h = {}

local p = {}
function p.main(frame)
    local args = util_args.merge()
    local popup = util_toggle.popupButton();
    popup.inner:node(h.getDisambigs(args[1]))
        :addClass('popup-disambig-container')
    return popup.button
end

function h.getDisambigs(player)
    local query = h.getQuery(player)
    return h.makeDisambigText(util_cargo.queryAndCast(query))
end

function h.getQuery(player)
    local query = {
        tables = 'Disambiguations',
        where = ('Term="%s"'):format(player),
        fields = "FinalLocation",
    }
    return query
end

function h.makeDisambigText(data)
    local ul = mw.html.create('ul')
        :addClass('popup-disambig-list')
    for _, row in ipairs(data) do
        h.printRow(ul, row)
    end
    return ul
end

function h.printRow(ul, row)
    ul:tag('li')
        :addClass('popup-disambig')
        :wikitext(util_text.intLinkOrText(row.FinalLocation))
        :tag('div')
            :addClass('copy-button')
            :wikitext('')
end

return p
```

This is pretty straightforward.

## Constructing the pop-up
So far, I have the ability to add a clickable button and the ability to print a list of items associated to that button.
### Styling the pop-up
Eventually I will probably move the `.copy-button` class to a more generic location, but for now it's exclusively a part of disambiguations, so I left the CSS here, in `MediaWiki:Gadget-popupDisambigs.css`.
```css
.popup-disambig-container {
    padding-right:1.3em;
}

.popup-disambig-list {
    white-space:nowrap;
}

.popup-disambig {
    position:relative;
}

.copy-button::after {
    content:'\f0c5';
    font-family: fontAwesome;
    position:absolute;
    right:-1em;
    top:0;
    display:block;
    cursor:pointer;
}
```
### Making the copy buttons do something
Here is the JS that makes the copy buttons work. Again, for the most part this is pretty generic and could certainly be made even more so, but since I haven't needed it for anything else yet, it's still just written for disambiguations.
```js
mw.hook('wikipage.content').add(function() {
    $('.popup-disambig .copy-button').click(function(e) {
        e.preventDefault();
        e.stopPropagation();
        var $el = $(this);
        var text = $el.parent().find('a').text();
        var copyEl = document.createElement('textarea');
        copyEl.value = text.replace(/_/g,' ');
        document.body.appendChild(copyEl);
        copyEl.select();
        document.execCommand('copy');
        document.body.removeChild(copyEl);
        document.execCommand('copy');
        $el.css('color','green');
        setTimeout(function() {
            $el.css('color','');
        }, 2000);
    });
});
```

### Replacing the ::after pseudo-element
Now to the fun part!

#### Pre-existing popup button code
This part deserves its own post; I'll do that eventually, but for now, here's the code for reference. If you aren't looking to implement this yourself, don't worry about this; just assume that I already have the ability to insert parsed wikitext into a popup button given some HTML constraints and an attribute containing the wikitext to parse.

I've included the full code at the end of this article, but here's an abridged version:
```js
window.popupButton = function(e) {
    e.stopPropagation();
    var hiddenClass = $(this).attr('data-toggler-hiddenclass');
    hiddenClass = hiddenClass ? hiddenClass : 'popup-content-hidden';
    // if we're re-clicking the same one we just showed before then, just hide it
    // otherwise hide all the others and show this one
    var $shownContent = $(this).find('.popup-content-wrapper-action:not(.' + hiddenClass + ')');
    if ($shownContent.length) {
        $shownContent.addClass(hiddenClass);
        return;
    }
    // hide everything...
    $('.popup-content-wrapper-action').addClass(hiddenClass);
    var $wrapper = $(this).find('.popup-content-wrapper-action');
    // ...and then show only this one
    $wrapper.toggleClass(hiddenClass);

    // snip
    // a lot of logic for resizing and moving stuff goes here
    
    $(document).click(function(){
        $('.popup-content-wrapper-action').addClass(hiddenClass);
    });
}
```

#### Constructing the pop-up HTML

Typically, I construct the necessary HTML structure in Lua as part of page creation, but remember I can't do that in this case, again because a user cannot be required to do anything special to show these pop-ups. So instead, I do this in JavaScript.

The strategy is as follows:
1. Determine if the click was in fact on top of the `::after` (we discussed this earlier).
2. If not, then just do a normal click event (i.e. go to the link target).
3. Determine the `Term` for this disambig.
4. Call the Lua Module `DisambigPopup` through the api `parse` action.
5. Insert the contents *after* the link. Note that this is actually changing the DOM in a relevant way, but we need to make it look to the user like nothing has happened. This is the fun part. Note that while we're inserting a visible action button, the tooltip itself is still hidden by a `display:none;` at this point in time.
6. Fire the hook needed to bind the click events for the copy buttons and for the pop-up itself.
7. Click the pop-up button, showing the tooltip content that we had previously inserted.

```js
$(function() {
    $('.catlink-disambiguations').click(function(e) {
        var el = this;
        if (e.pageX <= $(el).offset().left + $(el).innerWidth()) return;
        e.preventDefault();
        var disambigTitle = $(this).attr('title').replace(' (Disambiguation)', '');
        console.log(disambigTitle);
        return new mw.Api().get({
            action: 'parse',
            prop: 'text',
            disablelimitreport: 1,
            disableeditsection: 1,
            text: '{{#invoke:DisambigPopup|main|' + disambigTitle + '}}'
        }).then(function(data) {
            var wikitext = data.parse.text['*'];
            $(el).addClass('catlink-disambiguations-activated');
            $(el).wrap('<span class="disambig-link-wrapper"></span>');
            var newEl = document.createElement('span');
            $(newEl).html($(wikitext).html());
            $(newEl).insertAfter(el);
            
            // snip
            
            mw.hook('wikipage.content').fire($('.popup-button-action, .copy-button'));
            popupButton.bind(newEl)(new MouseEvent('click'));
            // snip
        });
    });
});
```

The snipped bits are to do with the extension [TippingOver](https://github.com/oOeyes/TippingOver), which I use to generate my tooltips. I'll show that at the end.

To accomplish the user-can't-see-anything part, we add the class `catlink-disambiguations-activated` in this function, and adjust our earlier CSS as follows:

```css
.catlink-disambiguations {
    position:relative;
}

.catlink-disambiguations:not(.catlink-disambiguations-activated) {
    margin-right:1em;
}

.catlink-disambiguations:not(.catlink-disambiguations-activated)::after {
    font-family:"FontAwesome";
    content:"\f05a";
    cursor:pointer;
    color:var(--body-text-color);
    position:absolute;
    right:-1em;
}

.disambig-link-wrapper {
    position:relative;
}

.disambig-link-wrapper a + span {
  width: 1em;
  display: inline-block;
  text-align: right;
}

.disambig-link-wrapper a + span .popup-button-pretty {
  text-align:left;
}
```

Note the additions of the `:not(.catlink-disambiguations-activated)` and the new styles at the end for the situation where we have the popup button separated from the link. Once we add the `activated` class, the `::after` will disappear completely, and the new element will remain in its exact place. Yay!

#### Enabling tooltips in the pop-up

TippingOver doesn't hook into the API `parse` method at all. So we have to do some of its work on our own. Here's the snipped bits from the earlier JS:

```js
            // tipping over doesn't hook into api parse action it seems
            // so, let's do it ourselves!
            // delete this eventually if this changes
            $(newEl).find('a').each(function() {
                var title = $(this).attr('title').replace(/ /g, '_');
                // we need to escape dashes that were previously in the title
                // but NOT escape dashes that we add
                // so instead of directly (\%\w\w) -> _$1- we'll use #_$1# as an intermediate step
                // then we do the -, (, ) replacements which aren't handled by encodeURIComponent
                // echoes encodeAllSpecial from https://github.com/oOeyes/TippingOver/blob/8f1dd018e57f284cc33db5cd1dbc967d24621fbf/includes/WikiTooltips.php
                var escapedTitle = encodeURIComponent(title)
                    .replace(/\%(\w\w)/g, '#_$1#')
                    .replace('-', '_2d-')
                    .replace('(', '_28-')
                    .replace(')', '_29-')
                    .replace(/\#(_\w\w)\#/g, '$1-');
                // Tooltip id for Ziv (Chen Yi) looks like: Ziv__28-Chen_Yi_29- 
                $(this).addClass('to_hasTooltip')
                    .attr('data-to-id', escapedTitle)
                    .attr('data-to-titles', title + '||' + 'Tooltip:' + title)
                    .attr('data-to-flags', 'fiem');
                $(this).removeAttr('title');
            });
            // end delete this eventually

            mw.hook('wikipage.content').fire($('.popup-button-action, .copy-button'));
            popupButton.bind(newEl)(new MouseEvent('click'));
            window.toWikiTooltips.beginInitialize(); // put tooltips into the pop-up
```

## Conclusion
Since I snipped a lot of code and edited a bunch of things and showed some things more than once, here's a full list of files. The links on the wiki may change significantly in the future, so I'm pasting the relevant code here as well.

* [popupDisambigs.js](https://lol.gamepedia.com/MediaWiki:Gadget-popupDisambigs.js)
* [popupDisambigs.css](https://lol.gamepedia.com/MediaWiki:Gadget-popupDisambigs.css)
* [highlightDisambigs.css](https://lol.gamepedia.com/MediaWiki:Gadget-highlightDisambigs.css)
* [toggles.css](https://lol.gamepedia.com/MediaWiki:Gadget-toggles.css) - for the popup functionality
* [toggles.js](https://lol.gamepedia.com/MediaWiki:Gadget-toggles.js) - for the popup styling
* [DisambigPage](https://lol.gamepedia.com/Module:DisambigPage)
* [DisambigPopup](https://lol.gamepedia.com/Module:DisambigPopup)
* [ToggleUtil](https://lol.gamepedia.com/Module:ToggleUtil)

After this is just code, so I'll conclude now - in the end this is probably about average for the amount of moving parts and effort that go into a "single" feature like this for me. As I mentioned before, I'm not totally happy with the current Cargo setup because of how restrictive it is in terms of having different types of disambiguations allowed, but in practice it's really not a big problem (over 90% of our disambiguated pages are for players anyway), and I mostly want to fix it because I care about doing things the right way (hopefully that's not too hard to believe after reading this post...).

While copying this entire setup might seem a bit daunting, the first part - just the highlighting of disambiguations, with no pop-ups - is already extremely valuable to have and requires only one extension - Highlight Links In Category. The one required CSS rule can be added to `common.css` (and also Mobile if you use Mobile Frontend), so not even Gadgets is required (though you [should](https://river.me/blog/gadgets-for-code-organization/)). And if you're feeling slightly ambitious, you can build something very similar to this entire setup with the only needed variation being your own `DisambigPage` module!

## Code files

popupDisambigs.js:
```js
// <nowiki>
$(function() {
    $('.catlink-disambiguations').click(function(e) {
        var el = this;
        if (e.pageX <= $(el).offset().left + $(el).innerWidth()) return;
        e.preventDefault();
        var disambigTitle = $(this).attr('title').replace(' (Disambiguation)', '');
        console.log(disambigTitle);
        return new mw.Api().get({
            action: 'parse',
            prop: 'text',
            disablelimitreport: 1,
            disableeditsection: 1,
            text: '{{#invoke:DisambigPopup|main|' + disambigTitle + '}}'
        }).then(function(data) {
            var wikitext = data.parse.text['*'];
            $(el).addClass('catlink-disambiguations-activated');
            $(el).wrap('<span class="disambig-link-wrapper"></span>');
            var newEl = document.createElement('span');
            $(newEl).html($(wikitext).html());
            $(newEl).insertAfter(el);
            
            // tipping over doesn't hook into api parse action it seems
            // so, let's do it ourselves!
            // delete this eventually if this changes
            $(newEl).find('a').each(function() {
                var title = $(this).attr('title').replace(/ /g, '_');
                // we need to escape dashes that were previously in the title
                // but NOT escape dashes that we add
                // so instead of directly (\%\w\w) -> _$1- we'll use #_$1# as an intermediate step
                // then we do the -, (, ) replacements which aren't handled by encodeURIComponent
                // echoes encodeAllSpecial from https://github.com/oOeyes/TippingOver/blob/8f1dd018e57f284cc33db5cd1dbc967d24621fbf/includes/WikiTooltips.php
                var escapedTitle = encodeURIComponent(title)
                    .replace(/\%(\w\w)/g, '#_$1#')
                    .replace('-', '_2d-')
                    .replace('(', '_28-')
                    .replace(')', '_29-')
                    .replace(/\#(_\w\w)\#/g, '$1-');
                // Tooltip id for Ziv (Chen Yi) looks like: Ziv__28-Chen_Yi_29- 
                $(this).addClass('to_hasTooltip')
                    .attr('data-to-id', escapedTitle)
                    .attr('data-to-titles', title + '||' + 'Tooltip:' + title)
                    .attr('data-to-flags', 'fiem');
                $(this).removeAttr('title');
            });
            // end delete this eventually
            
            mw.hook('wikipage.content').fire($('.popup-button-action, .copy-button'));
            popupButton.bind(newEl)(new MouseEvent('click'));
            window.toWikiTooltips.beginInitialize(); // put tooltips into the pop-up
        });
    });
});

mw.hook('wikipage.content').add(function() {
    $('.popup-disambig .copy-button').click(function(e) {
        e.preventDefault();
        e.stopPropagation();
        var $el = $(this);
        var text = $el.parent().find('a').text();
        var copyEl = document.createElement('textarea');
        copyEl.value = text.replace(/_/g,' ');
        document.body.appendChild(copyEl);
        copyEl.select();
        document.execCommand('copy');
        document.body.removeChild(copyEl);
        document.execCommand('copy');
        $el.css('color','green');
        setTimeout(function() {
            $el.css('color','');
        }, 2000);
    });
});
// </nowiki>
```

popupDisambigs.css:
```css
.catlink-disambiguations {
    position:relative;
}

.catlink-disambiguations:not(.catlink-disambiguations-activated) {
    margin-right:1em;
}

.catlink-disambiguations:not(.catlink-disambiguations-activated)::after {
    font-family:"FontAwesome";
    content:"\f05a";
    cursor:pointer;
    color:var(--body-text-color);
    position:absolute;
    right:-1em;
}

.popup-disambig-container {
    padding-right:1.3em;
}

.popup-disambig-list {
    white-space:nowrap;
}

.popup-disambig {
    position:relative;
}

.copy-button::after {
    content:'\f0c5';
    font-family: fontAwesome;
    position:absolute;
    right:-1em;
    top:0;
    display:block;
    cursor:pointer;
}

.disambig-link-wrapper {
    position:relative;
}

.disambig-link-wrapper a + span {
  width: 1em;
  display: inline-block;
  text-align: right;
}

.disambig-link-wrapper a + span .popup-button-pretty {
  text-align:left;
}
```

highlightDisambigs.css:
```css
.catlink-disambiguations {
    color:orange!important;
    font-weight:bold;
}

.sbs-disambig .catlink-disambiguations {
 color:var(--link)!important; /* im so sorry */
 font-weight:normal;
}
```

toggles.css (the relevant part):
```css
.popup-content-outer{
    position: relative;
}
.popup-content-inner-pretty {
    border: 1px solid var(--interface-border);
    background: rgba(var(--interface-background-rgb), 0.95);
    border-radius: 3px;
    box-shadow: 0.25em 0.25em 2px var(--box-shadow-color);
    position: absolute;
    top: 0;
    z-index:99;
}
.popup-content-middle-wrapper {
    position:absolute;
    top:50%;
    left:50%;
}
```

toggles.js (the relevant part):
```js

window.popupButton = function(e) {
    e.stopPropagation();
    var hiddenClass = $(this).attr('data-toggler-hiddenclass');
    hiddenClass = hiddenClass ? hiddenClass : 'popup-content-hidden';
    // if we're re-clicking the same one we just showed before then, just hide it
    // otherwise hide all the others and show this one
    var $shownContent = $(this).find('.popup-content-wrapper-action:not(.' + hiddenClass + ')');
    if ($shownContent.length) {
        $shownContent.addClass(hiddenClass);
        return;
    }
    // hide everything...
    $('.popup-content-wrapper-action').addClass(hiddenClass);
    var $wrapper = $(this).find('.popup-content-wrapper-action');
    // ...and then show only this one
    $wrapper.toggleClass(hiddenClass);
    var $inner = $(this).find('.popup-content-inner-action');
    // this should be a decrease in width so that we don't have wasted extra whitespace
    $wrapper.css('width', $inner.outerWidth(true));
    
    // adjust positioning if needed, in the x-direction
    var width = $wrapper.outerWidth(true);
    var buttonHalfWidth = $(this).width() / 2;
    var leftCoord, totalWidth;
    var topCoord, totalHeight;
    // a window container will be specified for example on the front page when we want popup area to be
    // constrained more strictly than just the edge of the page.
    // because there might be a more immediate positioned parent of the popup button, we
    // don't use offsetParent() method here, and instead manually calculate offsets using coordinates
    // of the element itself (the button) and the parent we specified with .popup-window-container
    var windowContainer = $(this).closest('.popup-window-container');
    if (windowContainer) {
        var coords = $(this).offset();
        var parentCoords = $(windowContainer).offset();
        leftCoord = parseInt(coords.left) - parseInt(parentCoords.left);
        totalWidth = $(windowContainer).width();
        // may as well grab y values here too since they're conditional, we'll use them later though
        topCoord = parseInt(coords.top) - parseInt(parentCoords.top);
        totalHeight = $(windowContainer).height();
    }
    else {
        var coords = this.getBoundingClientRect();
        leftCoord = parseInt(coords.left);
        totalWidth = $(window).width();
        topCoord = parseInt(coords.top);
        totalHeight = $(window).height();
    }
    if (leftCoord + width + buttonHalfWidth > totalWidth) {
        // case 1: this thing is TOO BIG just put it all the way in the other side and let it overflow still
        if (width > totalWidth) {
            $wrapper.css('left', -1 * parseInt(leftCoord) + 'px');
        }
        // case 2: this thing is overflowing unless we put the right amount of space over to the left
        else {
            $wrapper.css('left', parseInt(totalWidth) - (parseInt(leftCoord) + parseInt(width)) + 'px');
        }
    }
    // case 3: no overflow we are happy
    else {
        $wrapper.css('left', '');
    }
    
    // adjust positioning if needed, in the y-direction
    var height = $inner.outerHeight(true);
    var buttonHalfHeight = $(this).height() / 2;
    if (topCoord + height + buttonHalfHeight > totalHeight) {
        if (height > totalHeight) {
            $wrapper.css('top', -1 * parseInt(topCoord) + 'px');
        }
        else {
            $wrapper.css('top', parseInt(totalHeight) - (parseInt(topCoord) + parseInt(height)) + 'px');
        }
    }
    else {
        $wrapper.css('top', '');
    }
    
    $(document).click(function(){
        $('.popup-content-wrapper-action').addClass(hiddenClass);
    });
}
```

DisambigPage:
```lua
local util_args = require('Module:ArgsUtil')
local util_cargo = require('Module:CargoUtil')
local util_esports = require("Module:EsportsUtil")
local util_map = require('Module:MapUtil')
local util_sentence = require("Module:SentenceUtil")
local util_table = require('Module:TableUtil')
local util_text = require("Module:TextUtil")
local i18n = require('Module:i18nUtil')

local m_team = require('Module:Team')
local m_region = require('Module:Region')
local m_role = require('Module:Role')
local m_country = require('Module:Country')
local util_vars = require('Module:VarsUtil')

local lang = mw.getLanguage('en')

local ORDER = {
    'players',
    'teams',
    'other'
}

local TITLE
local CARGO_INDEX = 0

local h = {}

local p = {}
function p.main(frame)
    local args = util_args.merge()
    i18n.init('DisambigPage')
    h.setTitle(args)
    
    local sections = {
        intro = i18n.print('DisambigPage', mw.site.siteName),
        players = h.getAndStorePlayerData(args),
        teams = h.getAndStoreTeamData(args),
        other = util_args.numberedArgsToTable(args, 'other'),
    }
    return h.makeOutput(sections)
end

function h.setTitle(args)
    -- strip out (Disambiguation) text if needed
    if args.title then
        TITLE = args.title
        return
    end
    TITLE = mw.title.getCurrentTitle().text:gsub(' %(Disambiguation%)', '')
end

-- players
function h.getAndStorePlayerData(args)
    local tbl = util_args.numberedArgsToTable(args, 'player')
    if not tbl then return nil end
    util_map.arrayInPlaceWithIndex(tbl, h.getPlayerData, args)
    h.storeCargo(tbl, 'Player')
    return util_map.inPlace(tbl, h.makePlayerLine)
end

function h.getPlayerData(player, i, args)
    if mw.title.makeTitle('', player).exists then
        return h.getPlayerDataFromCargo(player)
    end
    return h.getPlayerDataFromArgs(args, i)
end

function h.getPlayerDataFromCargo(player)
    local row = util_cargo.getOneRow(h.makePlayerQuery(player))
    row.IsFormer = not row.Team
    row.Team = row.Team or row.TeamLast
    row.CurrentName = row._pageName ~= lang:ucfirst(player) and row._pageName
    row.Name = player
    return row
end

function h.makePlayerQuery(player)
    return {
        tables = 'Players, PlayerRedirects=PR',
        join = 'Players._pageName=PR._pageName',
        fields = h.fields,
        groupBy = 'Players._pageName',
        where = ('PR.AllName = "%s"'):format(player),
        types = {
            IsSubstitute = 'boolean',
            IsLowercase = 'boolean'
        }
    }
end

h.fields = {
    "Players.Residency=Residency",
    "Players.Team",
    "Players.TeamLast",
    "Players.RoleLast",
    "Players.Country",
    "Players.IsSubstitute",
    "Players.IsLowercase",
    "Players._pageName",
}

function h.getPlayerDataFromArgs(args, i)
    local index = 'player' .. i
    local row = {
        Name = args[index],
        Residency = args[index .. 'res'],
        Country = args[index .. 'country'],
        Team = args[index .. 'team'],
        RoleLast = args[index .. 'role'],
        IsSubstitute = util_cargo.strToBool(args[index .. 'isSubstitute']),
        IsFormer = util_cargo.strToBool(args[index .. 'isFormer']),
        CurrentName = args[index .. 'currentName'],
        _pageName = args[index],
    }
    return row
end

function h.makePlayerLine(row)
    local replacements = {
        RESIDENCY = m_region.onlyimage(row.Residency),
        LINK = util_text.intLinkOrText(row.Name),
        NATIONALITY = row.Country and m_country.localization(row.Country) or '',
        FORMER = row.IsFormer and i18n.default('former') or '',
        ROLE = m_role.sentence(row.RoleLast, { sub = row.IsSubstitute }),
        TEAM = m_team.rightmediumlinked(row.Team),
        NOW_KNOWN_AS = row.CurrentName and i18n.default('nowKnownAs', row.CurrentName) or '',
    }
    return util_sentence.makeReplacements(i18n.default('sentencePlayer'), replacements)
end

function h.pageExistsField(name)
    return mw.title.new(name, '').exists
end

function h.storeCargo(tbl, tableType)
    util_map.rowsInPlace(tbl, h.storeOneCargoRow, tableType)
end

function h.storeOneCargoRow(row, tableType)
    CARGO_INDEX = CARGO_INDEX + 1
    util_cargo.store(h.getGeneralCargoData(row, tableType))
end

function h.getGeneralCargoData(row, disambigType)
    local data = {
        _table = 'Disambiguations',
        FinalLocation = row._pageName,
        Term = TITLE,
        DisambigType = disambigType,
        N_LineInPage = CARGO_INDEX,
        DisambigID = h.getDisambigId(),
    }
    return data
end

function h.getDisambigId()
    return ('%s_%s'):format(mw.title.getCurrentTitle().text, CARGO_INDEX)
end

-- teams
function h.getAndStoreTeamData(args)
    local tbl = util_args.numberedArgsToTable(args, 'team')
    if not tbl then return nil end
    util_map.arrayInPlaceWithIndex(tbl, h.getTeamData, args)
    h.storeCargo(tbl, 'Team')
    return util_map.inPlace(tbl, h.makeTeamLine)
end

function h.getTeamData(team, i, args)
    if mw.title.makeTitle('', team).exists then
        return h.getTeamDataFromCargo(team, args['team' .. i .. 'freetext'])
    end
    return h.getTeamDataFromArgs(args, i)
end

function h.getTeamDataFromCargo(team, freetext)
    local row = util_cargo.getOneRow(h.makeTeamQuery(team))
    row.freetext = freetext
    row.Location = row.TeamLocation or row.Location
    return row
end

function h.makeTeamQuery(team)
    local fields = {
        'Region',
        '_pageName',
        'Location',
        'TeamLocation',
        'IsDisbanded',
        'IsLowercase',
    }
    return {
        tables = 'Teams',
        fields = fields,
        where = ('_pageName="%s"'):format(team),
        groupBy = '_pageName',
        types = {
            IsDisbanded = 'boolean',
            IsLowercase = 'boolean',
        }
    }
end

function h.getTeamDataFromArgs(args, i)
    local index = 'team' .. i
    local row = {
        Region = args[index .. 'region'],
        Location = args[index .. 'location'],
        IsDisbanded = util_cargo.strToBool(args[index .. 'isDisbanded']),
        _pageName = args[index],
        freetext = args[index .. 'freetext'],
    }
    return row
end

function h.makeTeamLine(row)
    local replacements = {
        REGION = m_region.onlyimage(row.Region),
        LINK = util_text.intLinkOrText(h.getCasedLink(row._pageName, row.IsLowercase)),
        DISBANDED = row.IsDisbanded and i18n.default('disbanded') or '',
        COUNTRY = row.Location and m_country.localization(row.Location) or '',
        END = h.getTeamLineEnd(row),
    }
    return util_sentence.makeReplacements(i18n.default('sentenceTeam'), replacements)
end

function h.getCasedLink(link, isLowercase)
    if not isLowercase then return link end
    return util_text.lcfirst(link)
end

function h.getTeamLineEnd(row)
    if not row.IsDisbanded and not row.Location then
        return row.freetext or ''
    end
    return i18n.default('teamEndDefault')
end

-- output
function h.makeOutput(sections)
    local output = mw.html.create()
    output:wikitext(sections.intro)
    h.printLists(output, sections)
    return output
end

function h.printLists(output, sections)
    notfirst = false
    for _, v in ipairs(ORDER) do
        h.printOneList(output, sections[v], v, notfirst)
        notfirst = true
    end
end

function h.printOneList(output, sectionData, sectionName, notfirst)
    if not sectionData or not next(sectionData) then return end
    output:wikitext(h.getIntro(sectionName, notfirst))
    local ul = output:tag('ul')
    util_map.inPlace(sectionData, h.printOneListItems, ul)
end

function h.getIntro(sectionName, notfirst)
    return i18n.print(
        'intro_' .. sectionName,
        TITLE,
        i18n.print(notfirst and 'also_refer' or 'refer')
    )
end

function h.printOneListItems(line, ul)
    ul:tag('li'):wikitext(line)
end

return p
```

DisambigPopup:
```lua
local util_args = require('Module:ArgsUtil')
local util_cargo = require("Module:CargoUtil")
local util_html = require("Module:HtmlUtil")
local util_table = require("Module:TableUtil")
local util_text = require("Module:TextUtil")
local util_toggle = require("Module:ToggleUtil")

local h = {}

local p = {}
function p.main(frame)
    local args = util_args.merge()
    local popup = util_toggle.popupButton();
    popup.inner:node(h.getDisambigs(args[1]))
        :addClass('popup-disambig-container')
    return popup.button
end

function h.getDisambigs(player)
    local query = h.getQuery(player)
    return h.makeDisambigText(util_cargo.queryAndCast(query))
end

function h.getQuery(player)
    local query = {
        tables = 'Disambiguations',
        where = ('Term="%s"'):format(player),
        fields = "FinalLocation",
    }
    return query
end

function h.makeDisambigText(data)
    local ul = mw.html.create('ul')
        :addClass('popup-disambig-list')
    for _, row in ipairs(data) do
        h.printRow(ul, row)
    end
    return ul
end

function h.printRow(ul, row)
    ul:tag('li')
        :addClass('popup-disambig')
        :wikitext(util_text.intLinkOrText(row.FinalLocation))
        :tag('div')
            :addClass('copy-button')
            :wikitext('')
end

return p
```

ToggleUtil (the relevant part):
```lua
-- popup button
-- all popup buttons are pretty, but we have the choice of what actions to additionally bind
-- by optionally calling the default popupButton or popupButtonLazy
function p.popupButton(tbl, class)
    -- default popup button
    local popup = p.popupButtonPretty(tbl, class)
    popup.button:addClass('popup-button-action')
    return popup
end

function p.popupButtonLazy(tbl, class, parse)
    -- content inside will be computed lazily
    -- the js adds {{}} for you, so only provide the part inside that
    local button = p.popupButton(tbl, class)
    button.button:addClass('popup-button-lazy')
        :attr('data-parse-text', parse)
    return button
end

function p.popupButtonPretty(tbl, class)
    -- does NOT add a button action, can be called directly or via p.popupButton
    -- p.popupButton fetches this then adds the button action and returns
    -- class can be specified to add custom popup-content, popup-button, and popup-wrapper class names
    if not tbl then tbl = mw.html.create() end
    if not settings then settings = {} end
    local span = tbl:tag('span')
    local button = span:tag('div')
        :addClass('popup-button-pretty')
        :wikitext('')
    local wrapper = button:tag('div')
        :addClass('popup-content-wrapper-pretty')
        :addClass('popup-content-wrapper-action')
        :addClass('popup-content-hidden')
    local inner = wrapper:tag('div')
        :addClass('popup-content-inner-pretty')
        :addClass('popup-content-inner-action')
    if class then
        inner:addClass('popup-content-' .. class)
        button:addClass('popup-button-' .. class)
        wrapper:addClass('popup-wrapper-' .. class)
    end
    return {
        tbl = tbl,
        span = span,
        button = button,
        wrapper = wrapper,
        inner = inner
    }
end
```