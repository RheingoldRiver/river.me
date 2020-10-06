---
title: "Top Schedule"
date: 2020-10-06T13:25:44Z
draft: false
tags:
    - leaguepedia
    - mediawiki
summary: How I created a global site banner on Leaguepedia using the extension Header Footer and some JavaScript, but no api calls
description: How I created a global site banner on Leaguepedia using the extension Header Footer and some JavaScript, but no api calls
---

Go to any page on [Leaguepedia](https://lol.gamepedia.com/League_of_Legends_Esports_Wiki), and you will see a banner along the top of the wiki showing a schedule upcoming matches with countdowns, links to the events, the teams playing, and streams. If there are too many matches scheduled concurrently then the code collapses them into a single block that just says "4 matches" or "16 matches" so that this block doesn't take up the entire screen.
![The front page of Leaguepedia](/images/top-schedule/frontpage.png)
Creating this display is [just a single Lua module](https://lol.gamepedia.com/Module:TopSchedule) (plus of course the entire Cargo infrastructure that it relies on); the more interesting part, and topic of this blog post, is how it gets to where it is - this part of the page isn't typically made available to you for editing by MediaWiki.

{{% leaguepedia_citation %}}

## Method 1.0 - Cargo query on each page load
### The supporting code
Originally, I created a Cargo table called `TopSchedule`. This table contained pre-rendered HTML that I processed in a Lua module, also called `TopSchedule`. The Lua module queried the next 50 upcoming matches from my `MatchSchedule` table, did some parsing for each one, and then stored a separate row into the Cargo table for each match. The module was invoked on a page called `Project:Top Schedule`, and blank edited by a crontab every 15 minutes.

The parsing isn't important, but here's the store that the Lua module did (this was so long ago that I didn't have `util_cargo.store` written yet!):
```lua
		frame:callParserFunction{
			name = '#cargo_store',
			args = {
				'',
				_table = 'TopSchedule',
				Time = row.DateTime,
				Display = val,
				UniqueLine = mw.title.getCurrentTitle().text .. '_' .. i
			}
		}
```
### The gadget
Here's the code that generated the display at the top of the page. It queries the Cargo table `TopSchedule`, un-escapes the HTML, and then puts the result after `#mw-head`.
```js
$.when( mw.loader.using( 'mediawiki.api' ), $.ready ).then( function () { 
	a = new mw.Api();
	a.get({
		action:'cargoquery',
		limit:'15',
		tables:'TopSchedule',
		fields:'Display',
		where:'Time > NOW() - INTERVAL 1 HOUR',
		order_by:'Time ASC',
		group_by: 'UniqueLine'
	}).done(function(data) {
		var text = []
		for (item in data.cargoquery) {
			text.push(data.cargoquery[item].title.Display);
		}
		var display = document.createElement('div');
		display.className = 'topschedule';
		display.innerHTML = text.join('').replace(/\&lt\;/g,'<').replace(/\&gt\;/g,'>').replace(/\&quot\;/g,'"');
		document.getElementById("mw-head").appendChild(display);
		mw.hook('wikipage.content').fire($('topschedule'));
	}).fail(function (error, code){
		console.log(code);
	});
});
```

This method lasted about three weeks - it was uncacheable and not performant AT ALL. Not good.

## Method 1.5 - get query on each page load
The appeal of the Cargo query was that I was always inserting precisely the right number of matches - I wanted 15 to show, I was querying 15, it was always 15 upcoming matches. But I clearly needed something cacheable, which meant that I needed a pre-rendered entire page. So I ditched the `TopSchedule` Cargo table and instead, in `Module:TopSchedule`, built my actual output: an entire array of 50 matches, not 15, tagged each one with a `data-i` attribute as well as its start time, and wrote some JS to hide all of the matches that had already started, as well as all of the matches after the first 15.

(Why 50? This is a rather arbitrary number, it's almost certainly more than I need, as I have a crontab refreshing the page every fifteen minutes, but it's performant enough, and this way I can guarantee I'll never run out of matches to show even if say 20 matches start at the same time, or if something stops my crontab from connecting for a couple hours.)

I've omitted the Lua code this time, but `Project:Top Schedule` now contained the exact contents of what I want to display as the top schedule - no more, no less.

Here's the JS; the majority of the code handles hiding all but the correct 15 matches:
```js
$.when( mw.loader.using( 'mediawiki.api' ), $.ready ).then( function() { 
    $.get('/index.php?title=Project:Top_Schedule&action=render').done(function(data) {
        $(data).appendTo(document.getElementById("mw-head"));
        var i = 0;
        $('.topschedule-box').each(function() {
        	var nowTime = Date.now();
        	var expTime = parseInt($(this).attr('data-expiration')) * 1000;
			if (nowTime >= expTime) {
				$(this).css('display', 'none');
				i = parseInt($(this).attr('data-i'));
			}
			else if (i + 15 < parseInt($(this).attr('data-i'))) {
				$(this).css('display', 'none');
			}
        })
        mw.hook('wikipage.content').fire($('topschedule'));
    }).fail(function (error, code){
        console.log(code);
    });
});
```

This method was fine, and it was in place for a bit over a year. But I wanted to do better, and eliminate the API query altogether! 

## Method 2.0 - Extension HeaderFooter
There's this lovely extension called [Header Footer](https://www.mediawiki.org/wiki/Extension:Header_Footer) that I'm now using for my top schedule. But, it's not quite that simple because nothing on my wikis ever is. Sooooo here's how to set up a global site header using Header Footer and how to reposition it to actually be a *site* header instead of a page header!

### Making the header global
Header Footer allows for per-namespace customization of messages, which we actually don't want in this case, so we'll need to "overwrite" the specific behavior into a global banner.

You need to create the pages `MediaWiki:Hf-nsheader-User` for each namespace. You can just do this by hand, but that sucks, so if you install my `river_mwclient` library (available [on GitHub](https://github.com/RheingoldRiver/river_mwclient)), which wraps/extends `mwclient` with a bunch of utilities, and set up your credentials as described in the README there, you can run the following code:

```python
from river_mwclient.wiki_client import WikiClient
from river_mwclient.auth_credentials import AuthCredentials

credentials = AuthCredentials(user_file="me")
site = WikiClient('https://wiki-url.com', path='/', credentials=credentials)

TEXT_TO_SAVE = '{{Project:Top Schedule}}'

for namespace in site.namespaces:
	page_name = 'MediaWiki:Hf-nsheader-{}'.format(namespace.name)
	site.client.pages[page_name].save(TEXT_TO_SAVE)
```
Of course, replace `TEXT_TO_SAVE` with whatever you actually want your banner to be.

Note, the standard library `mwclient`'s built-in `site.namespaces` only gives you the namespace number, and we need namespace name, so I'm using my own `site.namespaces`. If it weren't that annoying by default, I'd just give you code without custom dependencies; if you don't want to download `river_mwclient` then just do a `site.api` query of your own. Here's the code that generates my `site.namespaces` (I also have a `Namespace` class defined elsewhere in the library):

```python
    @property
    def namespaces(self):
        if self._namespaces is not None:
            return self._namespaces
        result = self.client.api('query', meta='siteinfo', siprop="namespaces|namespacealiases")
        ns_aliases = {}
        for alias in result['query']['namespacealiases']:
            alias_key = str(alias['id'])
            if alias_key not in ns_aliases:
                ns_aliases[alias_key] = []
            ns_aliases[alias_key].append(alias['*'])
        ret = []
        for ns_str, ns_data in result['query']['namespaces'].items():
            ns = int(ns_str)
            ret.append(Namespace(id_number=ns, name=ns_data['*'],
                                 canonical_name=ns_data.get('canonical'), aliases=ns_aliases.get(ns_str)))
        self._namespaces = ret
        return ret
```

### Making the header a header
The second part of this is turning the header into a **site** header instead of just a **page** header.

![The default location of a header](/images/top-schedule/default-location.png)
In this screenshot, I've made a page with the text "Hello world" and a header that says "Here is the default header location" - as you can see, the header is just part of the content area, which is not what we want at all. So we need to move it outside of the content area and to the top of the entire webpage, after `#mw-head`.

For this, we use javascript:
```js
$(function() {
	var $schedule = $('#top-schedule');
	if (! $schedule) return;
	$schedule.detach();
	$("#mw-head").append($schedule);
	$schedule.css('display', '');
});
```

So my Lua puts the entire top schedule into a div with id `top-schedule` and **inline** style `display:none;`. That way when it's loaded into the page at first, it's not flashed to the user in the wrong place. If it doesn't exist on the page (e.g. some special pages), nothing happens. Otherwise, I detach it from the wrong location, append it after `#mw-head`, and then remove the `display:none;`. And that's it, we're done!

Actually, the full version of the gadget needs to keep the bit that deals with showing only 15 entries, but your site banner probably won't need all that, so this is sufficient. But for good measure, here's my current version of [topSchedule.js](https://lol.gamepedia.com/MediaWiki:Gadget-topSchedule.js):

```js
$(function() {
	var $schedule = $('#top-schedule');
	if (! $schedule) return;
	$schedule.detach();
	$("#mw-head").append($schedule);
	$schedule.css('display', '');
    var i = 0;
    $('.topschedule-box').each(function() {
    	var nowTime = Date.now();
    	var expTime = parseInt($(this).attr('data-expiration')) * 1000;
		if (nowTime >= expTime) {
			$(this).css('display', 'none');
			i = parseInt($(this).attr('data-i'));
		}
		else if (i + 15 < parseInt($(this).attr('data-i'))) {
			$(this).css('display', 'none');
		}
    });
});
```

I put the above code in a gadget:

```
* topSchedule[ResourceLoader|default|hidden|targets=desktop]|topSchedule.js
```

Note in particular that this is not enabled in mobile, and if you're putting this in your `common.js` instead of making it a gadget, you should not put it in `mobile.js`, however [I do recommend using gadgets for code organization](https://river.me/blog/gadgets-for-code-organization/).

## Conclusion
To make a global site banner, you need:
* Extension Header Footer
* Create your banner on [some project page](https://lol.gamepedia.com/Leaguepedia:Top_Schedule), with an inline `display:none;` style
* Use a Python script to create transclusions of that page to the MediaWiki hf page for every namespace on your wiki (or do this by hand)
* Write a gadget (or just a function in common.js) to detach your banner and place it after `#mw-head` on page load

If applicable, you may need a crontab to regularly purge cache/blank edit the page your banner is saved on.