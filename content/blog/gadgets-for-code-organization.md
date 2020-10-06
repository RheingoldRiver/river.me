---
title: "Gadgets for code organization"
date: 2020-03-23T01:20:40Z
draft: false
tags:
    - mediawiki
    - javascript
    - css
summary: Though it's mostly used as a way to let wikis have opt-in or opt-out CSS and JS features for users, the Gadgets extension is a great way to organize code instead of using Common.css and Common.js for everything.
---

The [Gadgets extension](https://www.mediawiki.org/wiki/Extension:Gadgets) provides a means by which you can have opt-in or opt-out CSS and JavaScript for users to toggle in their preferences. But its utility extends beyond that as a method of code organization for wikis with large amount of CSS/JS.

{{% leaguepedia_citation %}}

Typically, all CSS and JS gets dumped into `common.css` and `common.js`, maybe also `mobile.css` and `mobile.js`, and finally maybe also one or two skin-specific files - on the Gamepedia platform, it's `hydradark.css` or `hydra.css`. Beyond that, users can choose to curate their own code files at `Special:MyPage/common.css` and corresponding additional pages, but that's about it. If you have thousands of lines of code, well, good luck organizing it -- you're probably going to make a bunch of pretty comments-as-headings and rely on ctrl+F to edit the huge documents you have open.

Enter the Gadgets extension. Gadgets effectively allow you to have any number of CSS and JS files, at `MediaWiki:Gadget-gadgetName.css` and `MediaWiki:Gadget-gadgetName.js`, etc. And by using the `hidden` and `default` parameters, you can override the "users can opt in or out" functionality of it, so users don't accidentally go to their preferences and disable your site's core display properties.

## How does it work?
Gadget definitions are curated at the page `MediaWiki:Gadgets-definition`. Gadgets are sorted into sections prefaced by `h2` headings. Each `h2` heading should be the key to a system message that displays an i18n string for that gadget. For example, you might have a section that starts:
```
== edit-gadgets ==
```
And then `MediaWiki:Gadget-section-edit-gadgets` (i.e. `MediaWiki:Gadget-section-` followed by the heading string) contains the text `Edit Gadgets
`. You can look up gadget examples on any major MediaWiki installation, including Wikipedia, to see how it works.

A typical gadget definition probably looks something like this:
```
* wikEd[ResourceLoader]|wikEd.js|wikEd.css
```

The name of the gadget is `wikEd`, and there's a JS file and a CSS file. `[ResourceLoader]` marks the gadgets as compatible with ResourceLoader. To implement this gadget definition, you make a page called `MediaWiki:Gadget-wikEd` with an i18n string to show in user preferences briefly explaining what the gadget is. The JS code all goes at `MediaWiki:wikEd.js`, and CSS goes at `MediaWiki:wikEd.css`. Now the CSS and JS will be enabled at the same time or disabled at the same time, according to the user's preference about whether they want wikEd as a package to be enabled.

In theory, if we wanted to make `wikEd` always available with no way to opt out, we could move the code to `MediaWiki:Common.css` and `MediaWiki:Common.js`. But `wikEd` is a ton of third-party code, and we definitely don't want to have to load it every time we load `Common.js`, so let's see how to accomplish this while leaving it as a gadget:
```
* wikEd[ResourceLoader|default|hidden]|wikEd.js|wikEd.css
```
This is now an equivalent situation to the end user as this code being in our common files!

But wait, we can do better than this - now say you create 15 different files of CSS, with properties grouped together in a deliberate and sensible way. You want to take advantage of gadgets' ability to organize your code, so you create one line per gadget with the name of the gadget and a css file, right?

Actually, no. There's no limit to the number of code files you can include in a single line. We can in fact do the following:

```
* modularStyles[ResourceLoader|default|hidden]|gcdNotificationSignup.css|predictions.css|topSchedule.css|championFlashcards.css|flairs.css|gameinfo.css|rosterswapPortal.css|scoreboards.css
```

If you think about a gadget as a collection of files a user can enable or disable, this seems like a rather large number of files to be here. But if we think of this as a catch-all list of all of the modular CSS files we want to create and load, this allows us to have a much more compact way to include them all.

There's two more things we should add:

```
* modularStyles[ResourceLoader|default|hidden|targets=desktop,mobile|type=styles]|gcdNotificationSignup.css|predictions.css|topSchedule.css|championFlashcards.css|flairs.css|gameinfo.css|rosterswapPortal.css|scoreboards.css
```
The two parameters we added are:
* `|targets=desktop, mobile` - this loads the gadget on both desktop AND mobile skins - no more copy-pasting everything from `common.css` to `mobile.css` like MobileFrontend makes you do!
* `|type=styles` - as long as the gadget contains ONLY css, this will cause the files to be loaded earlier - to quote the documentation, "Use `styles` for modules that only modify styling for elements already on the page (e.g. when customizing the skin, layout, or article content). It will cause the CSS files of the module to be included from the page HTML instead of being loaded via JavaScript."

## Maintaining code as gadgets
Once you have some gadgets set up, you can use the interface at `Special:Gadgets` to see all of your gadgets along with all of the files included, and i18n strings substituted in (if you see a page name surrounded by angle brackets, that means you need to create that i18n string in the `MediaWiki:` namespace).

![Screenshot of view at Special:Gadgets on Leaguepedia](/images/gadgets-for-code-organization/special-gadgets.png)

But, personally, I don't like to use this interface at all; I'd much rather see the direct source of what I created in the compact view at `MediaWiki:Gadgets-definition`.

![Screenshot of view at MediaWiki:Gadgets-definition on Leaguepedia](/images/gadgets-for-code-organization/gadgets-definition.png)

But, by default those hyperlinks that let you click directly to the pages where you can edit code don't exist. Of course, I did what I always do in situations like this and wrote some JavaScript to fix it:

```js
$(function() {
    if (mw.config.get('wgPageName') != 'MediaWiki:Gadgets-definition') return;
    if (window.location.href.indexOf("history") > 0) return;
    var urlPrefix = mw.config.get('wgServer') + '/MediaWiki:Gadget-';
    function replaceWithLink(str) {
        var link = document.createElement('a');
        $(link).attr('href', urlPrefix + str);
        $(link).html(str);
        return link.outerHTML;
    }
    $('#mw-content-text li').each(function() {
        var html = $(this).html();
        var htmlParts = html.split('|');
        for (i in htmlParts) {
            if (htmlParts[i].endsWith('css') || htmlParts[i].endsWith('js')) {
                htmlParts[i] = replaceWithLink(htmlParts[i]);
            }
        }
        var text = htmlParts.join('|');
        var firstPart = text.match(/^([^\|\[]*)/)[0];
        if (firstPart) text = text.replace(firstPart, replaceWithLink(firstPart));
        $(this).html(text);
    });
    $('#mw-content-text h2 .mw-headline').each(function() {
        $(this).html(replaceWithLink('section-' + $(this).html()));
    });
});
```

On the page `MediaWiki:Gadgets-definition` (but not when viewing the history), this code replaces each gadget name with a link to that gadget, and the result is that you can use the given interface view to navigate to individual gadget pages.

Finally, what about searching? Even though it sucks to have a single file with *everything* in it, it can't be denied that sometimes you're going to forget where you put a definition of a CSS property, or where was that place you used `expandtemplates` so you can copy the api parameter names. You're gonna need *some* way to search all of your code - fortunately, there is a very easy way! Just do a search restricted to the MediaWiki namespace for the code you're looking for. You can either do a normal search and then edit the namespaces you want to see in the `Advanced` tab, or you can make a keyword-search bookmark to do the restriction for you....More on that later....

Update - On Septeber 8, I [finally published](https://river.me/blog/url-bar-is-a-cli/) my "more on that later" post!