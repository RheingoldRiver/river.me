---
title: "Popups"
date: 2020-05-18T20:21:07Z
draft: true
tags:
    - mediawiki
    - javascript
    - css
    - mediawiki_api
summary: Lazily loading content can allow you to offer users a wide range of dynamic content options on a page without problematically increasing loading times. Here's how I've built a framework for both static and lazily-loaded popups.
---

[Two posts ago](https://river.me/blog/disambig-highlight/), I talked about my setup for automatically showing popups next to links to disambiguation pages. That entire structure relies on my previously-written "popup content" framework, so this time we'll discuss that.

Here's an example of a static popup:
![A pop-up in a navbox](/images/popups/menu-nav.png)

And here's an example of a lazily-loaded popup:
![Pop-up standings on the wiki front page](/images/popups/standings-frontpage.png)

{{% leaguepedia_citation_ema_help %}}

## History
Click [here](#creating-the-current-approach) to skip this section.

The first popup content I displayed on the wiki was called "tournament-team history," and it showed a single team's schedule of games from a tournament inside of a pop-up available in tournament results sections. If you were on SK Telecom T1's page and clicked the "tth" pop-up in the LCK 2019 Summer row, you'd see a table showing all of SK Telecom T1's games during Worlds 2019:

![SK Telecom T1's LCK 2020 Summer schedule](/images/popups/skt-lck.png)

I wrote this implementation in January 2019, around when I was finishing my tournament data project. At the time, I made everything with non-specific class names in CSS, but there wasn't really any general-case Lua. Over time I started creating more and more instances of pop-up content, and the demands I had of the system grew. I gradually refactored and redesigned, and about a year after I first created my tournament-team history toggle, I'd deleted all of its code completely, replaced with calls to general-case toggle functionality.

### Original CSS
```css
.popup-button {
	position:relative;
	display: inline-block;
}

.standings .popup-button {
	float:right;
	padding-left:.4em;
}

.achievements-lastresult-outer .popup-button {
	position:absolute;
	right:0;
	top: 50%;
	margin-top: -0.5em;
	line-height: 1;
}

.popup-button::before {
	font-family:"FontAwesome";
	content:"\f05a";
	cursor:pointer;
}

.popup-content {
	position:absolute;
	top:50%;
	left:50%;
	z-index:100;
	box-shadow: 0.25em 0.25em 2px var(--box-shadow-color);
	border-radius: 2px;
	border: 1px solid #000;
	background: #fff;
}

.popup-right .popup-content {
	right:50%;
	left:auto;
}

.popup-content .wikitable{
	margin:0;
}

.popup-content-hidden {
	display:none!important;
}

.popup-content-tth th,
.popup-content .standings .teamname,
.popup-content .standings {
	white-space:nowrap;
}

.popup-content .standings {
	margin-bottom:0;
}
```
### Original JavaScript
At the time, I didn't yet know the "hack" of wrapping the entire gadget in commented `<nowiki></nowiki>` tags to avoid script errors, wanted templates/categories, etc.

Note the unfortunate hard-coding of wikitext to feed to the `parse` api in this function.
```js
$( function () {
	
	$('.tournament-team-history').click(function(e){
		e.stopPropagation();
		var $this = $(this);
		var display = '.tournament-team-history-display';
		var hiddenclass = 'tournament-team-history-hidden';
		var $thisDisplay = $this.find(display);
		if ($thisDisplay.length !== 0 ) {
			if($($thisDisplay[0]).hasClass(hiddenclass)) {
				$(display).addClass(hiddenclass);
				$thisDisplay.removeClass(hiddenclass);
			}
			else {
				$thisDisplay.toggleClass(hiddenclass);
			}
			return;
		}
		$(display).addClass(hiddenclass);
		var team = $this.attr('data-tth-team');
		var page = $this.attr('data-tth-page');
		var display = $this.attr('data-tth-display');
		// please leave the string separated like this because otherwise there's a fake script error on this page
		var text = "{{" + "MatchListTournamentTeam|page=" + page + "|team=" + team + "|display=" + display + "}}";
		var a = new mw.Api();
		a.get({
			action : "parse",
			text : text,
			prop : "text"
		}).then(function(data){
			var result = data.parse.text["*"];
			result = result.replace('\\"','"');
			var div = document.createElement("div");
			var $div = $(div);
			$div.addClass('tournament-team-history-display');
			div.innerHTML = result;
			$this.append(div);
			$(document).click(function(){
				$('.tournament-team-history-display').addClass('tournament-team-history-hidden');
			});
		},function(code, data){
			console.log(code);
			return;
		});
	});
});
```

### Original Lua
Here's the Lua that created the HTML (later on I'll include examples of generated HTML as well, but since this code isn't live anymore, I don't have that here):
```lua
local util_args = require('Module:ArgsUtil')

local h = {}

local p = {}
function p.main(frame)
	local args = util_args.merge(true)
	if args.type == 'standings' then
		return p.standings(nil, args.page)
	elseif args.type == 'tth' then
		return p.tth(nil, args.link, args.display, args.team)
	end
end

function p.tth(tbl, link, display, team)
	if not tbl then tbl = mw.html.create() end
	tbl:tag('div')
		:addClass('popup-button') -- general styling for outer table
		:addClass('popup-button-tth') -- for js
		:attr('data-tth-team', team)
		:attr('data-tth-page', link)
		:attr('data-tth-display', display)
	return tbl
end

return p
```

Like I mentioned in the JS section, there's no data attributes indicating the code that should be run by the `parse` command, so that's hard-coded into the JS, resulting in a bad separation of information. And the "action" class names added here are all specific to this one setting; I was using the generic `.popup-button` to create the display but TTH-specific classes for the JS. As I created more and more code using these, this quickly became a problematic approach not just in theory but also in practice; many different popups needed to be able to load content lazily, and the content to display was always different.

At the end of January 2019, I added a new section to `Module:PopupButton`:

```lua
function p.standings(tbl, page, right)
	if not tbl then tbl = mw.html.create() end
	local div = tbl:tag('div')
		:addClass('popup-button') -- for styling
		:addClass('popup-button-standings') -- for js
		:attr('data-standings-page', page)
	if right then
		div:addClass('popup-right')
	end
	return tbl
end
```
As soon as I created this separate function (and its corresponding JS, which was basically a copy-paste of the TTH JS), I knew I'd need to abstract and make something generic.

## Creating the current approach
I'll now give up on the historical aspect (it took a lot of checking version history and reading my [2019 dev blog notes](https://lol.gamepedia.com/Leaguepedia:Dev_Blog/2019) to reconstruct the first section), and talk about my current approach.

### HTML
#### Lua code
One issue that arose early on was that I had needed classes for CSS that were also used in my JavaScript, so if I wanted to reuse CSS, I was necessarily reusing JS as well. But I want to have a range of available behaviors for my pop-ups, so this was a problem. My solution was to redo all my classes and split them so that every element had two: one with a suffix of `-pretty`, which was used exclusively for CSS, and one with a suffix of `-action`, used for JS. 

If a non-default behavior - such as lazy loading - is desired, the standard `-action` classes can just not be added, by calling `p.popupButtonPretty` directly.

Here's the resulting module code to build the HTML. In both cases, a table of HTML objects is returned so that additional classes can be added as needed. (There's a third case shown later on for lazy loading)
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
#### Example output
Here's the generated HTML for a simple pop-up. This is part of a navbox. (The comments are mine and edited in for this post, not generated by the wiki.)

```html
<div class="popup-button-pretty popup-button-action">
	<div class="popup-content-wrapper-pretty popup-content-wrapper-action popup-content-hidden">
		<div class="popup-content-inner-pretty popup-content-inner-action navbox-event-list">
			<!-- Begin content - this is not part of the popup code and could be anything -->
			<ul>
				<li><a href="/LCS/2020_Season" class="to_hasTooltip" title="LCS/2020 Season" data-to-id="LCS_2f-2020_Season" data-to-titles="LCS/2020 Season||Tooltip:LCS/2020 Season" data-to-flags="fieM">Overview</a></li>
				<li><a href="/LCS/2020_Season/Spring_Season" class="to_hasTooltip" title="LCS/2020 Season/Spring Season" data-to-id="LCS_2f-2020_Season_2f-Spring_Season" data-to-titles="LCS/2020 Season/Spring Season||Tooltip:LCS/2020 Season/Spring Season" data-to-flags="fieM">Spring Season</a></li>
				<li><a href="/LCS/2020_Season/Spring_Playoffs" class="to_hasTooltip" title="LCS/2020 Season/Spring Playoffs" data-to-id="LCS_2f-2020_Season_2f-Spring_Playoffs" data-to-titles="LCS/2020 Season/Spring Playoffs||Tooltip:LCS/2020 Season/Spring Playoffs" data-to-flags="fieM">Spring Playoffs</a></li>
				<li><a href="/LCS/2020_Season/Summer_Season" class="to_hasTooltip" title="LCS/2020 Season/Summer Season" data-to-id="LCS_2f-2020_Season_2f-Summer_Season" data-to-titles="LCS/2020 Season/Summer Season||Tooltip:LCS/2020 Season/Summer Season" data-to-flags="fieM">Summer Season</a></li>
				<li><a href="/LCS/2020_Season/Summer_Playoffs" class="to_hasTooltip" title="LCS/2020 Season/Summer Playoffs" data-to-id="LCS_2f-2020_Season_2f-Summer_Playoffs" data-to-titles="LCS/2020 Season/Summer Playoffs||Tooltip:LCS/2020 Season/Summer Playoffs" data-to-flags="fieM">Summer Playoffs</a></li>
			</ul>
			<!-- End content -->
		</div>
	</div>
</div>
```

### JavaScript
#### Max-width
In the above Lua, you might be wondering, what's with the `.popup-content-inner` and `popup-content-wrapper` nodes both being there? Shouldn't I just have a single absolutely-positioned `div` inside of the button `span`?

The problem with having only two layers is max width. Because of the absolute positioning of `.popup-content-wrapper-pretty` within `.popup-button-pretty`, the content inside of `.popup-inner-pretty` is made as small a width as possible. This results in one of the three cases being guaranteed to happen:
1. Add a `white-space:nowrap;` on literally everything (but what if I want it to wrap at 300px?)
2. Add a `width:` on literally everything (but what if I have less than X pixels of content in side of some of the pop-ups? `max-width` does not work in this case, it still forces all of the content as narrow as possible)
3. Just put up with the fact that everything's going to kind of suck

I didn't like any of those options. So what I did was to create an artificial `max-width` property myself in the following way:
1. Create two nested divs inside of the button; the outer one has class `.popup-content-wrapper-pretty` (I'll call it the "wrapper"), and the inner one has class `.popup-content-inner-pretty` (I'll call it the "inner").
2. Put a fixed width on the wrapper
3. On click, get the `outerWidth()` of the inner element
4. Assign this width to the wrapper

This process is executed in the following code:
```js
	var $inner = $(this).find('.popup-content-inner-action');
	// this should be a decrease in width so that we don't have wasted extra whitespace
	$wrapper.css('width', $inner.outerWidth(true));
```

While this seems like a pretty straightforward thing to do, at the time it took me and Ema many, many hours of experimentation and failure, spread over several months, to come up with the idea to do it this way. And even after we came up with this solution, it still needed multiple tweaks before it worked correctly everywhere. But as of writing this post, the implementation shown here has been stable for about six months, so I think we finally got it right.

#### Positioning within the window
If a pop-up is going to "spill over" the edge of its containing element (which could be the window), we need to adjust its position. By default, we've been printing these so that the upper-left corner is located at the center of the button, but walls can shift this up or left, all the way up to the point at which it's tangent to the edge of its container on either the top or left wall; at that point, the remaining spill-over is allowed.

Like the max-width fix, I wrote this code over a period of several months, but this was much less of an ordeal because there was never a case of not knowing even the general idea of an approach to fix any issue; the challenge was simply implementing fixes correctly. Here were some of the issues I encountered:
* At first I was checking left-right overflow only because it was unlikely that top-bottom would be an issue in any of my use cases, but of course eventually it did become a problem, so then I had to add vertical position-changing as well.
* Originally it only detected window borders for adjusting positioning, but then I had an issue on the front page where I had sections that I wanted `overflow:scroll;`, and that was creating an equally annoying situation to popups falling off the edges of the window, so I added support for a class `.popup-window-container`.
* For a while, I was using `offset` (return coordinates relative to entire page) instead of `getBoundingClientRect` (return coordinates relative to current window) in the entire-window case; this took me a couple months to notice (and never got reported).
* The arithmetic to figure out the proper repositioning was also just messy to figure out.
* Technically there's STILL an edge case that I'm not considering, which is when a popup is encountering on one side the edge of a `.popup-window-container` and on the opposite, the edge of the current window; in this case, the code below respects the former but *not* the latter; however, because `.popup-window-container`s are designed to be relatively small, so far I've decided this isn't worth it to fix.

When I started writing this section I realized anything I might say about how it works should in fact be a comment directly in the code. So, see comments for what it does:
```js
	// time to compute dimensions so we can shift this stuff if needed
	// we don't want pop-ups to overflow out of window/bounding box provided by page
	// we're doing all this before we actually show the thing, so we can guarantee there's no "jump"
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
	if (windowContainer.length > 0) {
		// in this case use offset which respects the position of the elements wrt entire document
		// in theory we should probably support position within window also, but
		// for the most part, this situation should only occur with relatively small areas
		// e.g. front page sections
		// so to make things SIGNIFICANTLY simpler, don't worry about maybe spilling off window
		var coords = $(this).offset();
		var parentCoords = $(windowContainer).offset();
		leftCoord = parseInt(coords.left) - parseInt(parentCoords.left);
		totalWidth = $(windowContainer).width();
		// may as well grab y values here too since they're conditional, we'll use them later though
		topCoord = parseInt(coords.top) - parseInt(parentCoords.top);
		totalHeight = $(windowContainer).height();
	}
	else {
		// getBoundingClientRect in order to get position within current window
		// we want to avoid spilling off window
		// n.b. this implies avoiding spilling off document as well
		var coords = this.getBoundingClientRect();
		leftCoord = parseInt(coords.left);
		totalWidth = $(window).width();
		topCoord = parseInt(coords.top);
		totalHeight = $(window).height();
	}
	
	// now that we know all of the relevant dimensions, do the adjustments
	
	// first the x-direction if needed
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
	
	// now the y-direction, again if needed
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
```

### CSS
There's a bunch of extra CSS beyond this in the gadget, but this is the styling common to all popups, though of course sometimes part of this may be overridden, particularly the choice of FontAwesome icon for the button. You'll also see here a comment restating the explanation above of why I have both the wrapper and inner elements. The full code in this gadget at the time of writing is provided at the end of the post.

You can see the live version of this code at [MediaWiki:Gadget-popups.css](https://lol.gamepedia.com/MediaWiki:Gadget-popups.css) on Leaguepedia. Note the use of css variables for every color; this is how we support both light and dark skins at the same time.
```css
/* js can be found at gadget-toggles.js */

.popup-content-hidden {
	display:none!important;
}

.popup-button-pretty {
	position:relative;
	display: inline-block;
}

.popup-button-pretty::before {
	font-family:"FontAwesome";
	content:"\f05a";
	cursor:pointer;
}

/* popup-content-wrapper needs to exist because it's impossible to attach a max-width without a width to popup-conent and have it act normally, instead the content will try to compress as narrow as possible because it's a child of popup-button. So we attach a width to popup-content-wrapper and then popup-content will be constrained by that but otherwise won't take up any extra space. All of the visual styles are attached to popup-content, so we don't know that there's a fixed-width div containing it. But to make sure we don't accidentally create an unnecessary horizontal scrollbar in some situations, we can still adjust the width of popup-content-wrapper in the same JS that shows it on click. */

.popup-content-wrapper-pretty {
	position:absolute;
	top:50%;
	left:50%;
}

.popup-content-pretty {
	z-index:100;
	box-shadow: 0.25em 0.25em 2px var(--box-shadow-color);
	border-radius: 3px;
	border: 1px solid var(--interface-border);
	background: rgba(var(--interface-background-rgb), 0.95);
	position: absolute;
	top: 0;
	z-index:99;
}

.popup-content-pretty .wikitable {
	margin:0;
}

/* navboxes */
.navbox .popup-button-pretty {
	margin-left:.5em;
}

.navbox-event-list {
	white-space:nowrap;
	padding-right:.5em;
}

.navbox .hlist .popup-button-pretty::before {
 content:'\f0c9';
  font-size:80%;
}
```

## Displaying dynamic (lazy) content
The process has two steps:
1. Insert the dynamic content into the `popup-content-inner-action`. This is done by reading the contents of the `data-parse-text` attribute on the node with class `popup-button-lazy`, wrapping it in `{{}}`, and then feeding that result to the api `parse` module.[^parseapi] After doing a bit of escaping, we get HTML code that we can add directly into the `.popup-content-inner-action` node.
1. Call `popupButton()`, the handler of a normal pop-up button, which will reposition and display the content as discussed earlier.
[^parseapi]: The parameter `disablelimitreport` removes a comment with some metadata about the parse, and `disableeditsection` removes any links to "edit" from page headings. Any time you're inserting text into a page via the `parse` action, you probably want both of these set.

Since this reduces to a case of normal pop-ups in the end, there's no new CSS to discuss here. As above, we'll first look at the HTML and its generation, and then the JS code.
### HTML
#### Example output
As you can see, there's no content inside of the inner div currently. The text that will be run by the wiki later is provided as the attribute `data-parse-text` on the button.
```html
<div class="popup-button-pretty popup-button-tth popup-button-action popup-button-lazy" data-parse-text="MatchListTournamentTeam|page=2020 Mid-Season Cup|team=DAMWON Gaming|display=DWG Schedule">
	<div class="popup-content-wrapper-pretty popup-content-wrapper-action popup-content-hidden popup-wrapper-tth">
		<div class="popup-content-inner-pretty popup-content-inner-action popup-content-tth"></div>
	</div>
</div>
```
#### Lua code
This code is part of the same `Module:ToggleUtil` shown above, and reuses the funtion `p.popupButtonPretty()`. You can see the full code at the end of this post.
```lua
function p.popupButtonLazy(tbl, class, parse)
	-- content inside will be computed lazily
	-- the js adds {{}} for you, so only provide the part inside that
	local button = p.popupButton(tbl, class)
	button.button:addClass('popup-button-lazy')
		:attr('data-parse-text', parse)
	return button
end
```
### JavaScript
We start by removing any event handler on the button in case a hook is re-fired at some point on the page; without this removal, we'd end up with two duplicate event handlers on click action. Both would happen, resulting in two API queries, the content added and replaced, and then the button being clicked twice - i.e. shown and then re-hidden, and in fact nothing visible to the user happens. Maybe this isn't the cleanest solution to this issue, but it works.

This is also a good thing to remember when debugging - if your click event that's supposed to toggle show-hide looks like it is doing nothing, and you're getting no errors, inspect the element and see how many click event handlers it has assigned! If it's even, there's a good chance you're firing some hook a second (or further) time and rebinding the same event twice.
```js
$('.popup-button-lazy').off('click');
$('.popup-button-lazy').click(function(e) {
	e.preventDefault();
	e.stopPropagation();
	var $this = $(this);
	var button = this;
	var text = '{{' + $this.attr('data-parse-text') + '}}';
	new mw.Api().get({
		action : "parse",
		text : text,
		prop : "text",
		disablelimitreport: 1,
		disableeditsection: 1,
	}).then(function(data){
		var result = data.parse.text["*"];
		result = result.replace('\\"','"');
		var resultBody = $(result).html();
		$this.find('.popup-content-inner-action').html(resultBody);
		$this.off('click')
		$this.click(window.popupButton);
		return $.Deferred().resolve();
	}).then(function() {
		return window.popupButton.bind(button)(e);
	});
});
```
## Conclusion
That's it! The disambiguation pop-ups are "just" a special case of this, with also a bit of extra code to maintain the functionality of the `::after` click and maintaining visual consistency when the `::after` is replaced with a separate node.

## Full code
[MediaWiki:Gadget-toggles.js](https://lol.gamepedia.com/MediaWiki:Gadget-toggles.js):
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
	
	// time to compute dimensions so we can shift this stuff if needed
	// we don't want pop-ups to overflow out of window/bounding box provided by page
	// we're doing all this before we actually show the thing, so we can guarantee there's no "jump"
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
	if (windowContainer.length > 0) {
		// in this case use offset which respects the position of the elements wrt entire document
		// in theory we should probably support position within window also, but
		// for the most part, this situation should only occur with relatively small areas
		// e.g. front page sections
		// so to make things SIGNIFICANTLY simpler, don't worry about maybe spilling off window
		var coords = $(this).offset();
		var parentCoords = $(windowContainer).offset();
		leftCoord = parseInt(coords.left) - parseInt(parentCoords.left);
		totalWidth = $(windowContainer).width();
		// may as well grab y values here too since they're conditional, we'll use them later though
		topCoord = parseInt(coords.top) - parseInt(parentCoords.top);
		totalHeight = $(windowContainer).height();
	}
	else {
		// getBoundingClientRect in order to get position within current window
		// we want to avoid spilling off window
		// n.b. this implies avoiding spilling off document as well
		var coords = this.getBoundingClientRect();
		leftCoord = parseInt(coords.left);
		totalWidth = $(window).width();
		topCoord = parseInt(coords.top);
		totalHeight = $(window).height();
	}
	
	// now that we know all of the relevant dimensions, do the adjustments
	
	// first the x-direction if needed
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
	
	// now the y-direction, again if needed
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

$(function() {
	
	// snip a bunch of toggle-but-not-popups code
	
	// Popups
	
	// How to use:
	// Create toggles using Module:ToggleUtil
	// Define a unique class on the wrapper element and assign a width larger than the largest expected internal width
	// Then the wrapper's width will shrink to match the inner content's outer width
	// This can be used to guarantee either a max-width on the inner content when inner is variable width
	// (such as in game recap sentences)
	// Or to guarantee a full width on the fixed-width inner content with an arbitrarily large outer width value
	// (such as in tournament-team history)
	
	$('.popup-button-action').off('click');
	$('.popup-button-action').click(window.popupButton);
	
	$('.popup-button-lazy').off('click');
	$('.popup-button-lazy').click(function(e) {
		e.preventDefault();
		e.stopPropagation();
		var $this = $(this);
		var button = this;
		var text = '{{' + $this.attr('data-parse-text') + '}}';
		new mw.Api().get({
			action : "parse",
			text : text,
			prop : "text",
			disablelimitreport: 1,
			disableeditsection: 1,
		}).then(function(data){
			var result = data.parse.text["*"];
			result = result.replace('\\"','"');
			var resultBody = $(result).html();
			$this.find('.popup-content-inner-action').html(resultBody);
			$this.off('click')
			$this.click(window.popupButton);
			return $.Deferred().resolve();
		}).then(function() {
			return window.popupButton.bind(button)(e);
		});
	});
	
});
// </nowiki>
```

[MediaWiki:Gadget-popups.css](https://lol.gamepedia.com/MediaWiki:Gadget-popups.css):
```css
/* js can be found at gadget-toggles.js */

.popup-content-hidden {
	display:none!important;
}

.popup-button-pretty {
	position:relative;
	display: inline-block;
}

.popup-button-pretty::before {
	font-family:"FontAwesome";
	content:"\f05a";
	cursor:pointer;
}

/* popup-content-wrapper needs to exist because it's impossible to attach a max-width without a width to popup-conent and have it act normally, instead the content will try to compress as narrow as possible because it's a child of popup-button. So we attach a width to popup-content-wrapper and then popup-content will be constrained by that but otherwise won't take up any extra space. All of the visual styles are attached to popup-content, so we don't know that there's a fixed-width div containing it. But to make sure we don't accidentally create an unnecessary horizontal scrollbar in some situations, we can still adjust the width of popup-content-wrapper in the same JS that shows it on click. */

.popup-content-wrapper-pretty {
	position:absolute;
	top:50%;
	left:50%;
}

.popup-content-pretty {
	z-index:100;
	box-shadow: 0.25em 0.25em 2px var(--box-shadow-color);
	border-radius: 3px;
	border: 1px solid var(--interface-border);
	background: rgba(var(--interface-background-rgb), 0.95);
	position: absolute;
	top: 0;
	z-index:99;
}

.popup-content-pretty .wikitable {
	margin:0;
}

/* navboxes */
.navbox .popup-button-pretty {
	margin-left:.5em;
}

.navbox-event-list {
	white-space:nowrap;
	padding-right:.5em;
}

.navbox .hlist .popup-button-pretty::before {
 content:'\f0c9';
  font-size:80%;
}

/* standings */
.standings .popup-button-tth-pretty {
	float:right;
	padding-left:.4em;
}

.popup-content-tth td,
.popup-content-tth th,
.popup-content-tth .standings .teamname,
.popup-content-tth .standings,
.popup-content-tth-pretty th,
.popup-content-pretty .standings .teamname,
.popup-content-pretty .standings {
	white-space:nowrap;
}

.popup-content-pretty .standings {
	margin-bottom:0;
}

.popup-wrapper-tth {
 width:600px;
}

.standings .popup-button-pretty.popup-button-tth {
	float:right;
	padding-left:.4em;
}

.popup-button-pretty .popup-content-tth .wikitable,
.popup-button-pretty .popup-content-tth .wikitable2 {
	margin:-1px;
}

/* legends */
.legend-icon.popup-button-pretty .popup-content-pretty {
 z-index: 101;
}

.legend-icon.popup-button-pretty::before {
  content:'';
}

.legend-icon.popup-button-pretty {
  background-image:url("/media/thumb/2/26/Standings_Help_Icon_2.png/20px-Standings_Help_Icon_2.png");
  height:20px;
  width:20px;
  cursor:pointer;
}

/* chromas */
.popup-button.chroma-button::after {
	content:;
}

/* achievements */
.achievements-lastresult-outer .popup-button-pretty {
	position:absolute;
	right:0;
	top: 50%;
	margin-top: -0.5em;
	line-height: 1;
}

/* references */
.popup-ref-wrapper {
	width:30em;
}

.popup-ref-button::before {
	font-family:unset;
	content:'[ref]';
	color:var(--link);
}

.ref-start .popup-ref-button::before {
	font-family:unset;
	content:'[s]';
	color:var(--link);
}

.ref-end .popup-ref-button::before {
	font-family:unset;
	content:'[e]';
	color:var(--link);
}

.popup-ref-wrapper-inner {
 padding:.3em;
}
```

[Module:ToggleUtil](https://lol.gamepedia.com/Module:ToggleUtil):
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
