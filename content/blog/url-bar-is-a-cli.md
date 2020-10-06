---
title: "Your URL bar is a CLI"
date: 2020-09-08T02:32:49Z
draft: false
tags:
    - firefox
    - leaguepedia
summary: When working with wikis, you can use your URL bar as a CLI-like interface for navigation, making your life a lot easier and more efficient!
description: When working with wikis, you can use your URL bar as a CLI-like interface for navigation, making your life a lot easier and more efficient!
---
Back in March, I wrote [a post about gadgets](https://river.me/blog/gadgets-for-code-organization/) in which I briefly mentioned making keyword-search bookmarks with namespace-specific restrictions to be able to easily search for code, and concluded with, "More on that later...." Well, it took me six months to get to it but here's that post finally!
## A note on browser support
This article is about Firefox.

My original plan for this article was to do a quick search to see how to do the equivalent setup in Chrome and make it browser agnostic. However, that quick search [revealed](https://superuser.com/questions/540076/how-to-add-keyword-to-bookmark-in-chrome) that in fact the equivalent setup is drastically different - instead of editing bookmarks, you edit search engines, so if you use Chrome then all I'm going to is give this brief mention that you can probably do something similar in Chrome and leave you on your own to figure out the rest. Or switch to Firefox, support the open Internet.

Now, onto the cool stuff you can do.
## Direct keywords
![A bookmark with a keyword](/images/url-bar-is-a-cli/bookmark.png)
After you save a bookmark, you can go back to it, right-click, go to Properties, and then add a `Keyword`. I typically make the keyword the same as the name, which is to say that I make the name the same as the keyword; the keyword is the functional and important part. If you type the keyword into your url bar, it will send you to the bookmark.

For example, `lolfp` (short for "lol front page") is the keyword for the bookmark https://lol.gamepedia.com; when I type ctrl+L then `lolfp` I'm sent to my wiki's home page.

Nice, we can bookmark some things like:

| Name | Keyword | Target
|------|---------|--------
| lolfp | lolfp | https://lol.gamepedia.com
| lolrc | lolrc | https://lol.gamepedia.com/Special:RecentChanges
| lolcss | lolcss | https://lol.gamepedia.com/MediaWiki:Common.css
| lolcssme | lolcssme | https://lol.gamepedia.com/Special:MyPage/common.css
| loljs | loljs | https://lol.gamepedia.com/MediaWiki:common.js
| loljsme | loljsme | https://lol.gamepedia.com/Special:MyPage/common.js
| lolbp| lolbp | https://lol.gamepedia.com/Special:BotPasswords
| loldr | loldr | https://lol.gamepedia.com/Special:DoubleRedirects
| lolbr | lolbr | https://lol.gamepedia.com/Special:BrokenRedirects

## Single-substitution keywords

Let's go one step farther. Firefox allows you to make a substitution of a single input after your keyword into a `%s` in your target. For example, if your target is https://lol.gamepedia.com/index.php?search=%s&title=Special%3ASearch (note the `%s` after `search=`) and you type `lol Faker` then you will be redirected to https://lol.gamepedia.com/index.php?search=Faker&title=Special%3ASearch. (If you actually click this link, MediaWiki will then further redirect you to Faker's page, https://lol.gamepedia.com/Faker.)

Here are some useful examples:

| kw | Target | Comment
|---------|-------- |---------
| lol     | https://lol.gamepedia.com/index.php?search=%s&title=Special%3ASearch |
| loltest | https://lol.gamepedia.com/Special:MyPage/%s | Easy sandbox pages!
| urmlol | https://lol.gamepedia.com/Special:UserRights/%s | This violates my conventions of starting with "lol" but is what I used historically
| lolct | https://lol.gamepedia.com/Special:CargoTables/%s | omitting any argument after `lolct` takes you just to Special:CargoTables
| lolcd | https://lol.gamepedia.com/Template:%s/CargoDec | Discipline with naming conventions on-wiki is important so your keywords work!!!
| lolsm | https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns828=1 | insource, in module ns
| lolsmw | https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns8=1 | insource, in mediawiki ns
| lolst | https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns10=1 | insource, in template ns
| lolsd | https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns10008=1 | insource, in data ns
| lolmwd | https://lol.gamepedia.com/MediaWiki:%s | `d` is for `direct` (as opposed to insource), maybe this should be `loldmw`
| lolt | https://lol.gamepedia.com/Template:%s | this should really be `loltd` (or `loldt`)

These module, MediaWiki, template, and data namespace search keywords are what I was talking about back in March when I mentioned namespace-specific searches for looking for code.

## Multiple-substitution keywords
Now say you want to make more than one substitution in your URL, for example you want to search an arbitrary string in an arbitrary wiki - useful in particular if you work with a very large wiki farm, such as I do with Gamepedia. Then you need to fill in both the domain of the wiki and also the search string. This is possible, but it requires use of a JavaScript bookmarklet. It'll look something like this:

![JavaScript bookmarklet screenshot](/images/url-bar-is-a-cli/js-bookmark.png)
The full bookmark in the screenshot is as follows:

```
javascript:%20%20var%20s='%s';%20url='https://%s.gamepedia.com/index.php?search=%s&title=Special:Search';%20tt=[''];%20j=0;%20qc=0;%20chunks=url.split('%s');%20for(i=0;%20i<s.length;%20i++){%20if(s.charAt(i)=='%22')qc=qc^1;%20if%20(s.charAt(i)=='%20'&&(qc||j>=chunks.length-2))%20{%20tt[j]+=('^');%20}%20else%20if%20(s.charAt(i)=='%20'){%20j++;%20tt.push('%20');%20}%20else%20{%20tt[j]+=(s.charAt(i));%20}%20}%20t%20=%20tt.join('');%20args=t.split(/\s/);%20nurl='';%20for(i=0;%20i<chunks.length;%20i++){%20nurl+=chunks[i];%20%20if(args[i]!=undefined)%20{%20args[i]=args[i].replace(/\^/g,'%20');%20nurl+=args[i];%20}%20}%20location.replace(nurl,'<\%20BR>');
```

This bookmarklet script is code that I adapted from [this LifeHacker article from 2007](https://lifehacker.com/firefox-tip-how-to-set-up-multi-parameter-keyword-sear-240552), my version improves at least one important thing - if the **last parameter** in your substitution contains spaces, they will automatically be bundled together without the need for quotation marks. I can't remember if I've made any other changes, it's possible but I don't think so.

Here's the un-minified code if you're curious:

```js
var s = "%s";
url = "https://%s.gamepedia.com/index.php?search=%s&title=Special:Search";
tt = [""];
j = 0;
qc = 0;
chunks = url.split("%s");
for (i = 0; i < s.length; i++) {
	if (s.charAt(i) == "%22") qc = qc ^ 1;
	if (s.charAt(i) == " " && (qc || j >= chunks.length - 2)) {
		tt[j] += "^";
	} else if (s.charAt(i) == " ") {
		j++;
		tt.push(" ");
	} else {
		tt[j] += s.charAt(i);
	}
}
t = tt.join("");
args = t.split(/\s/);
nurl = "";
for (i = 0; i < chunks.length; i++) {
	nurl += chunks[i];
	if (args[i] != undefined) {
		args[i] = args[i].replace(/\^/g, " ");
		nurl += args[i];
	}
}
location.replace(nurl, "< BR>");
```

Actually, [ever since Firefox 67](https://bugzilla.mozilla.org/show_bug.cgi?id=1553377), this bookmarklet method is needed even if you have only one `%s`, if that `%s` is in the domain of your URL. So before this patch, `%s.gamepedia.com/Special:BotPasswords` could be a normal single-substitution bookmark, but now it has to be a JavaScript bookmarklet. (That was a *fun* issue to solve back when I did that version upgrade.)

### Why not a "true" CLI though?
If I'm already using JavaScript, you could imagine I might do something like the following:
```
lol --ns 828 --insource term
lol -n 828 -s term
lol --nsname data --intitle term
lol -m data -t term
lol -m data -t term1 -s term2
```
Why don't I?
* This is just way too verbose (and `-` is too awkward to press) to be practical, even with one-character abbreviations, 99% of the time what I have is completely sufficient and I don't need the extra power that I could get by implementing anything like this
* So as a result it would be worse UX most of the time
* Half my use cases wouldn't even fit into this (where does Special:BotPasswords go?)
* This sounds like way too much work compared to what I did to make my current setup happen which was almost nothing.

But often I think "man it would be so neat if..." but yeah I'm never going to do this lol.

Though I will say I've considered for a while having some kind of redirection of namespace name to namespace number because while I do know offhand that namespace 828 is module, that's really the only one I know by heart, and for `Special:AllPages`, remembering the rest of them would be extremely helpful...but the redirection is most likely something I'd put on-wiki, not in my browser.

## Updating these bookmarks

As you may have guessed, maintaining these bookmarks from within the Firefox UI is...not really feasible. Fortunately, you can do all your editing within HTML files and import them. Here's all my Leaguepedia bookmarks:

```html
<!DOCTYPE NETSCAPE-Bookmark-file-1>
<!-- Shortcuts for accessing useful Gamepedia pages. Can be imported to firefox. -->
<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=UTF-8">
<TITLE>Bookmarks</TITLE>
<H1>Bookmarks Menu</H1>

<DL><p>
	<DT><H3>Search Keywords - LoL</H3>
	<DL><p>
		<DT><A HREF="https://lol.gamepedia.com/Special:WhatLinksHere/%s" SHORTCUTURL="links">links</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=%s&title=Special%3ASearch" SHORTCUTURL="lol">lol</A>
		<DT><A HREF="https://lol-archive.gamepedia.com/index.php?search=%s&title=Special%3ASearch" SHORTCUTURL="lola">lola</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:Contributions/%s" SHORTCUTURL="lolcontribs">lolcontribs</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:BotPasswords" SHORTCUTURL="lolbp">lolbp</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=%s&title=Special:Search&profile=advanced&fulltext=1&ns10008=1" SHORTCUTURL="loldat">loldat</A>
		<DT><A HREF="https://lol.gamepedia.com/Module:CargoDeclare/%s" SHORTCUTURL="lolcdm">lolcdm</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:ApiSandbox" SHORTCUTURL="lolapi">lolapi</A>
		<DT><A HREF="https://lol.gamepedia.com" SHORTCUTURL="lolfp">lolfp</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?title=%s&action=history" SHORTCUTURL="lolh">lolh</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:DoubleRedirects" SHORTCUTURL="loldr">loldr</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:BrokenRedirects" SHORTCUTURL="lolbr">lolbr</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:%s" SHORTCUTURL="lolmwd">lolmwd</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:AllPages?namespace=%s" SHORTCUTURL="lolsap">lolsap</A>
		<DT><A HREF="https://lol.gamepedia.com/Module:Bracket/%s" SHORTCUTURL="lolb">lolb</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns8=1" SHORTCUTURL="lolsmw">lolsmw</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:Analytics" SHORTCUTURL="lolanal">lolanal</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MyPage/hydradark.css" SHORTCUTURL="lolhyddme">lolhyddme</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:hydradark.css" SHORTCUTURL="lolhydd">lolhydd</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:hydra.css" SHORTCUTURL="lolhyd">lolhyd</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MyPage/hydra.css" SHORTCUTURL="lolhydme">lolhydme</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:Gadgets-definition" SHORTCUTURL="lolgd">lolgd</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns10008=1" SHORTCUTURL="lolsd">lolsd</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:Gadget-%s" SHORTCUTURL="lolg">lolg</A>
		<DT><A HREF="https://lol.gamepedia.com/Category:Pages with script errors" SHORTCUTURL="lolse">lolse</A>
		<DT><A HREF="https://lol.gamepedia.com/Template:%s/CargoDec" SHORTCUTURL="lolcd">lolcd</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?title=%s&action=edit" SHORTCUTURL="lole">lole</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MyPage/common.js" SHORTCUTURL="loljsme">loljsme</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MyPage/common.css" SHORTCUTURL="lolcssme">lolcssme</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns10=1" SHORTCUTURL="lolst">lolst</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:common.css" SHORTCUTURL="lolcss">lolcss</A>
		<DT><A HREF="https://lol.gamepedia.com/MediaWiki:common.js" SHORTCUTURL="loljs">loljs</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special:Search&profile=advanced&fulltext=1&ns828=1" SHORTCUTURL="lolsm">lolsm</A>
		<DT><A HREF="https://lol.gamepedia.com/%s" SHORTCUTURL="lold">lold</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:RecentChanges" SHORTCUTURL="lolrc">lolrc</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:CargoTables/%s" SHORTCUTURL="lolct">lolct</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?title=%s&action=pagevalues" SHORTCUTURL="lolpv">lolpv</A>
		<DT><A HREF="https://lol.gamepedia.com/Module:%s" SHORTCUTURL="lolm">lolm</A>
		<DT><A HREF="https://lol.gamepedia.com/index.php?search=insource%3A%s&title=Special%3ASearch" SHORTCUTURL="lols">lols</A>
		<DT><A HREF="https://lol.gamepedia.com/Template:%s" SHORTCUTURL="lolt">lolt</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MyPage/%s" SHORTCUTURL="loltest">loltest</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:MovePage/%s" SHORTCUTURL="move">move</A>
		<DT><A HREF="https://lol.gamepedia.com/Special:UserRights/%s" SHORTCUTURL="urmlol">urmlol</A>
	</DL><p>
```
...........and [here's all my bookmarks, period](https://github.com/RheingoldRiver/misc_settings/blob/master/wiki_search_bookmarks_ff.html).

In my main bookmarks folder, these are always the bottom-most folders in the menu, like so:

![Screenshot of bookmarks folder](/images/url-bar-is-a-cli/bookmarks-folder.png)

When I make a change, I delete and then re-import all of the keyword bookmarks. The process to do this is:
* Go to Show All Bookmarks from the bookmarks menu (or ctrl+shift+B)
* Click once to focus the bookmarks folder from within the library browser
* Shift-click to highlight and then delete all the old folders
* Go to Import and Backup
* Import bookmarks from HTML...
* Re-import the edited HTML file

![Import bookmarks from HTML](/images/url-bar-is-a-cli/import.png)

And then everything is re-imported!

## Mass updating with regex

If you've looked at my repo linked above, you may notice I have...a lot of copies of extremely similar bookmarks. Obviously, I'm updating these with regex when I decide I want to add a new one. (Yes, that does mean I'm [editing HTML with regex](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags), shh, it's okay) Here are my regex:

Find:
```
^(\s*<DT><A HREF="https://([^.]*).gamepedia.com/Special:Contributions/%s" SHORTCUTURL="(.+?)contribs">(.+?)contribs</A>)
```

Replace:
```
$1\r\n\t\t<DT><A HREF="https://$2.gamepedia.com/Special:BotPasswords" SHORTCUTURL="$3bp">$3bp</A>
```
In this example, I'm adding the line with the bookmark with the `bp` keyword immediately after the the line with the bookmark with the `contribs` keyword for every single Gamepedia wiki in the HTML file.

Note that this regex somewhat annoyingly does NOT take care of the JS bookmarklets at the end (the Gamepedia Global folder), those have to be done separately, but that's okay, I only really add new URLs here once every couple months. Typically I tend to do them in batches; when I have 3 or 4 that I want to add I'll do them at once since the process of adding new ones is just barely annoying enough that I don't like to do it too frequently.

When I need to add a new wiki, I copy one of the existing wikis, find-replace its domain, and then paste it into the list in the appropriate place in the alphabet.

## Conclusion
For a small set of wikis, I can navigate through them extremely easily just by pressing ctrl+L and then typing individual-word keywords. Then, for the other 2000-ish wikis on the Gamepedia farm, I can use a very slightly more verbose syntax to perform the same navigations, just by typing the full domain of the wiki instead of an abbreviated form of it.

The setup to get here is relatively low tech, and depending on your use cases, you may want to use some of the same shortcuts I use or make up your own. Get creative, but try and use consistent conventions as much as possible! I'd love to hear what you come up with!