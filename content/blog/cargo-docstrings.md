---
title: "How to add Cargo docstrings on table pages"
date: 2020-02-10T04:16:35Z
tags:
    - mediawiki
    - cargo
    - lua
    - javascript
draft: false
summary: This is not a post about how to add Cargo docstrings on table pages. Rather, it's a post about how to create workarounds in MediaWiki in a reasonably well-controlled manner.
---
This is not a post about how to add Cargo docstrings on table pages. Rather, it's a post about how to create workarounds in MediaWiki in a reasonably well-controlled manner.

{{% leaguepedia_citation %}}

Requirements:
* Scribunto (Lua)
* Cargo (obviously)
* JavaScript (recommended to use Gadgets but you can just put it in `common.js`)

Step 1 is to create a wrapper for the built-in functionality that isn't working precisely as desired. In this case, we'll create a wrapper for the parser function `cargo_declare`. Our wrapper doesn't need to do much: 
1. Accept a table of key-value pairs, the keys are Cargo field names, and the values are types (or similar structure)
1. Accept a table name
1. Return a valid `cargo_declare`

The following structure follows this layout:
```lua
return {
    Date_Display = 'String',
    Date_Sort = 'String',
}
```

But since we want to include a description as well, we'll adjust to have multiple values associated to each key:
```lua
return {
    Date_Display = { type = "String", desc = "" },
    Date_Sort = { type = "DateTime", desc = "" },
}
```

And since the order of parameters in `#cargo_declare` is important (so we can control the order of fields), we might prefer like so:
```lua
return {
    Date_Display = { type = "String", desc = "", order = 1 },
    Date_Sort = { type = "DateTime", desc = "", order = 2 },
}
```

But typing out `order` is kind of annoying, so we'll tweak one more time and instead of returning key-value pairs to access using `pairs()` we'll create an ordered list of tables to access using `ipairs()`. Here's the final format, with a few extra rows included:
```lua
return {
	{ field = "Date_Display", type = "String", desc = "Date for displaying in timelines, may contain \"approx\" etc" },
	{ field = "Date_Sort", type = "DateTime", desc = "Invisible to the reader, but must be an exact date. Can guess if the date is approximate, put qualifying information into Date_Display" },
	{ field = "IsApproxDate", type = "Boolean", desc = "If the date is approx or not - independent from how much precision to use as the display, this will just add the text (approx.)" },
	{ field = "EarliestPossibleDate", type = "Date", desc = "Based on values of approx/display date" },
	{ field = "LatestPossibleDate", type = "Date", desc = "Based on values of approx/display date" },
	{ field = "Sentence", type = "Wikitext", desc = "Display text (English-only)" },
	{ field = "SentenceWithDate", type = "Wikitext", desc = "Display text (English-only). Prefixed by display date." },
```

Note that the final transition we did isn't necessarily the best way to organize data - suppose, for example, that we want to reuse these declarations as a way to automatically add typing to fields when querying them; in this case, accessing via key would be more convenient than iterating through an array using `ipairs`. But since queries are expensive to do, we won't be doing many per page, so it's not a big problem to restructure the table dynamically as needed (probably into an ordered dictionary), and that will be done inside of a wrapper anyway, so we won't be increasing complexity. The most common use case for accessing these fields is going to be to adjust or create new declarations, so we want that to be very easy to interact with.

The important thing to note about what we've done so far is that none of this depends on the fact that we'll be using an unsupported method of attaching docstrings to table fields. If the syntax for `#cargo_declare` included a syntax for specifying a `description`, our table layout would be exactly the same. This is important for two reasons:
* We aren't adding additional "accidental" complexity by forcing editors to know which parts of a declaration parameter are "official" and which are custom.
* If Cargo ever adds native support for field docstrings/descriptions, we won't have to change any repeated-use code.

Now we have decided how we want to write our declarations, but there's no code yet to actually *do* anything. Let's change that.

The following snippet is originally found at [Module:CargoDeclare](https://lol.gamepedia.com/Module:CargoDeclare).
```lua
function p.declare(args)
    local tableName = args[1]
    local cargoArgs = h.concatFieldsToArgs(h.getArgs(tableName))
    return mw.getCurrentFrame():callParserFunction{
        name = ('#cargo_declare:_table=%s'):format(tableName),
        args = cargoArgs
    }
end

function h.getArgs(tableName)
    return require('Module:CargoDeclare/' .. tableName)
end

function h.concatFieldsToArgs(fields)
    util_map.inPlace(fields, h.concatFieldToArg)
    return fields
end
```

There's some additional make-it-available-from-wikitext wrapping to fetch args from frame as well as the definition of `util_map.inPlace()` (it does what you'd guess) but this is pretty much all that's needed for the declaration to work.

Beyond that, there's two places I want docstrings available. The first is on the actual Cargo Declare template page, as a table (additional examples in use can be seen [here](https://lol.gamepedia.com/Category:Cargo_Declaration_Templates)):
![Example Cargo documentation with docstrings](/images/cargo-docstrings/doc-table.png)
And code to generate this:
```lua
function p.doc(args)
    i18n.init('CargoDeclare')
    local tableName = args[1]
    return h.makeDocTable(h.getArgs(tableName))
end

function h.makeDocTable(fields)
    local output = mw.html.create('table')
        :addClass('wikitable')
    util_map.rowsInPlace(fields, h.formatForDocTable)
    util_html.printHeaderFromI18n(output, DOC_COLUMNS)
    util_html.printRowsByList(output, fields, DOC_COLUMNS)
    return output
end
```

The second is on the actual Cargo table page (additional examples in use [here](https://lol.gamepedia.com/Special:CargoTables/)):
![Example Special:CargoTables page with docstrings](/images/cargo-docstrings/cargo-page.png)
This part is a bit tricker since we can't actually control creation of this page, and here's where the "random hack" part of this entire thing comes in. Here's the JavaScript code:

```js
$(function() {
    var $cargoFields = $('.mw-special-CargoTables #mw-content-text > ul');
    if (!$cargoFields.length) return;
    var tableName = mw.config.get('wgTitle').match(/\/([^\/]+)$/)[1];
    if (!tableName) return;
    return new mw.Api().postWithToken('csrf', {
        action : 'expandtemplates',
        prop : 'wikitext',
        text : '{{#invoke:CargoDeclare|main|' + tableName + '|forgadget=yes}}'
    }).then(function(data) {
        var wikitext = data.expandtemplates.wikitext;
        var dict = wikitext.split(';;;');
        var lookup = {};
        dict.forEach(function(entry) {
            var tbl = entry.split(':::');
            lookup[tbl[0]] = tbl[1];
        });
        $cargoFields.find('li').each(function() {
            var field = $(this).find('strong').html();
            var el = document.createElement('span');
            if (lookup[field] !== undefined) {
                $(el).html(' - ' + lookup[field]);
            }
            this.appendChild(el);
        });
    });
});
```
In our `CargoDeclare` module, we have the following:
```lua
function p.forGadget(args)
    local cargoArgs = h.getArgs(args[1])
    return util_table.concat(cargoArgs, ';;;', h.concatForJson)
end

function h.concatForJson(field)
    return ('%s:::%s'):format(field.field, field.desc)
end
```
This will give us the `;;;`-separated list of parameters that we can add to the table. Probably the worst part of this (other than the fact that we're doing it in the first place?) is the selector at the start since Cargo doesn't provide any class names for us to use. But as you can see, the only code we'd have to actively change in the event of Cargo changing is contained in this one JavaScript snippet - there's some Lua too, but that could in theory just stick around and be unused. Crucially, *none of the places we are adding table definitions will have to be updated*.

To summarize, we've:
1. Defined a syntax to use to feed data to a wrapping module for `#cargo_declare`.
1. Written a wrapper that can parse this syntax to create:
    * The actual `#cargo_declare`
    * A table with documentation on the declaring template
    * A `;;;`-separated list of key-value pairs that we can access via JavaScript
1. Written a short JS snippet that can be used to import our key-value pairs to a `Special:CargoTables` page.

And we've done so in a way that's nicely encapsulated and can be easily changed at any point in time if needed!
