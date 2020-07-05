---
title: "Use cases for Highlight Links in Category"
date: 2020-06-30T23:21:53Z
draft: false
tags:
    - mediawiki
    - leaguepedia
summary: Highlight Links In Category is a very small MediaWiki extension that applies classes to links based on the links' targets' categories. It has a number of incredibly nice use cases; here I give some examples.
---
[Highlight Links in Category](https://www.mediawiki.org/wiki/Extension:Highlight_Links_in_Category) is a small MediaWiki extension, hosted on [GitHub](https://github.com/labster/HighlightLinksInCategory). All it does is apply classes to internal links based on their targets' categories, and a couple weeks ago I [wrote a patch](https://github.com/labster/HighlightLinksInCategory/pull/7/files) to follow links' redirect targets as well and pull categories from the target rather than the source. This extension is indispensible for my wiki, but I don't often see other wikis using it, so here I will go over some use cases, including a mix of some actual use cases of my own and some hypothetical ones.

## Disambiguations
I [wrote an entire post in May](https://river.me/blog/disambig-highlight/) about my usage of this extension to highlight disambiguations. At the time, the lack of redirect-following support was a fairly large pain point, but of course that's no longer an issue. While the entire setup detailed in that post is relatively involved, you can easily get a large percentage of the benefits with just three steps:
1. Add all disambiguations to a category, for example `Disambiguations`
2. Define `$wgHighlightLinksInCategory` so that your disambig category adds a class, for example `catlink-disambiguations`
3. Add a line of CSS to give these an incredibly obvious style, for example:
   ```css
   .catlink-disambiguations {
        color:orange!important;
        font-weight:bold;
    }
   ```

That's it! Now your links to disambiguations will be shown in bold orange text, making it extremely obvious to your editors that something is wrong.

Note the use of `!important` in this rule - this is actually a reasonable use case for it, because links have a lot of different styles, especially in MediaWiki, and we always want to override in this case.

If you only want this displayed to some users, e.g. administrators, you can take advantage of [user-group CSS](https://www.mediawiki.org/wiki/Manual:User_group_CSS_and_JavaScript) or [rights-governed gadgets](https://www.mediawiki.org/wiki/Extension:Gadgets) to apply the styles only to people likely to be able to fix the issues.

## Pop-up content
![Pop-up content - disambiguation links](/images/highlight-links-in-category/popup.png)
This is the final 20% of the disambiguations functionality from [my earlier post about disambig highlighting](https://river.me/blog/disambig-highlight/) that takes up 80% (really closer to 95% or more) of the effort. See that post for more details on how to accomplish this - but the short version is, you add an `::after` pseudoelement to all your links, add some click handling so that clicking the pseudoelement can bring up a popup instead of following the link, and then add a DOM element to replace the `::after` without any visual change to the user. Inside that DOM element, you can populate whatever information you like.

![Pop-up content - team schedule](/images/highlight-links-in-category/popup-other-info.png)
This process can be done for any kind of information that you're able to query about a page - you could show thumbnails of images here, or even display an entire infobox. Because you are loading the content lazily, it can be as indepth as you want without affecting overall wiki performance. It's a slightly more user-effort-required alternative to the extension [TippingOver](https://github.com/oOeyes/TippingOver). Depending on performance concerns and how much you want random pop-ups showing up any time a user moves their cursor around, you may prefer this approach over that of TippingOver. FWIW, I still use TippingOver in some contexts. One large advantage of the Highlight Links method is that you can click content in your tooltip to follow links, whereas TippingOver doesn't permit this since the tooltips avoid the cursor.

## Work in progress
![WIP - Almost done!](/images/highlight-links-in-category/wip-almost-done.png)
Are you in the middle of converting pages from one layout to another? Add them to a category when they are (or aren't) done, and apply styles accordingly! Shown is a green check in an `::after` pseudoelement. This gives administrators (or all users, your choice) an easy way to see at a glance what still needs to be done, and what's completed. This is especially useful when your conversion process has multiple components that must be completed sequentially, or some other kind of dependency issue - a line of green checks in a list can instantly convey, "this page is ready to change," whereas a red X says "don't change this yet, its dependencies aren't completed yet."

## Styling templates
Typically, I would advise the use of [Template:Tlx](https://www.mediawiki.org/wiki/Template:Tlx) or [Template:Tl](https://www.mediawiki.org/wiki/Template:Tl) to provide styling, but this is an illustrative example, and it could simplify your wikicode slightly so I'm including it as an option here anyway.

If users want to be able to link to template pages using the normal bracket syntax instead of one of the above templates, and you categorize all your templates, you can add surrounding braces by putting `{{` and `}}` into `::before` and `::after` pseudoelements, add a background color to the text, and display in a monospaced font.

Considering `tlx` and `tl` exist, and that "regular" users are unlikely to link to templates ever, this is a rather contrived example, but the point remains that you can draw attention to "meta" pages in a manner like this, with all of the styling provided by CSS instead of being inline.

I want to stress again that one of the advantages of this approach is to have special styling only for some user classes / user rights. This can allow you to draw your admins' attention to certain types of links without confusing the majority of users.

## Flairs
I mostly consider enterprise MediaWiki users as the audience of this blog, but all of my personal editing is on gaming or gaming-related wikis; this example is more relevant to the latter. Say you want to prefix every link to a playable character with one icon, and every link to an NPC (non-playable character) with a different one, to let people see easily at-a-glance which characters are available to them. The "typical" way to do this might be to create `Template:CharacterLink` which looks up the character's status in a lookup module or using Cargo or something, and adds the appropriate flair. But Highlight Links in Category lets you instead apply this flair as the background image of a `::before` pseudoelement and add this style to links without any special template invocation!

This is a HUGE improvement over the template method:
* The DOM isn't altered, as it would be if you were inserting a `File:` link or wrapping the link in a `span`.
* There's no burden of knowledge for users to understand how to use the `CharacterLink` template.
* There's no need to maintain the `CharacterLink` template.
* Needing to make requests for one-off pieces of data about another page is not a great requirement - you don't want to force one database call per link on the entire page, and more performant options either are significantly more complex (e.g. using the extension [LuaCache](https://github.com/HydraWiki/LuaCache) to wrap your Cargo/SMW queries) or force suboptimal data management (e.g. using a lookup module instead of Cargo/SMW). The Highlight Links in Category approach has none of these issues.
* One could also have usage like `{{CharacterLink|CharName|type=NPC}}` - this is a horrible single-source-of-truth violation, and subject to both burden-of-knowledge, convenience, and long-term maintenance problems.

## Stubs & maintenance
This use case is what prompted me to see if something like Highlight Links in Category existed in the first place - and in fact, I was going to write an extension to do this if I couldn't find one. If you create a large number of stub articles on your wiki (see our [anti-Thanos snap](https://river.me/blog/anti-thanos-april-fools/)), it might be helpful for users to know that these are "practically red links," but since the pages exist they won't actually be red links. You lose a lot of transparency on what articles need attention and what don't when you have pages like this; whereas a red link can scream "create me!" a stub article has no visual cue that it needs to be expanded.
![Low-content page links are shown in a different color](/images/highlight-links-in-category/low-content.png)
Enter, of course, Highlight Links in Category - add all your stub pages to a category (you can do this via a category attached to the `Stub` template), pick another color, and you're done! This can be extended to other maintenance issues - pages that need images, translation work, cleanup, etc.

Similar to the Disambiguations case, you probably want to use an `!important` in this rule.