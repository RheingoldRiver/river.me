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
2. Accept a table name
3. Return a valid `cargo_declare`

Actually, we will tweak the format of the table slightly; instead of key-value pairs, we'll give an ordered list of tables that have field name and field type. Since our goal is to attach a docstring/description to each field, we'll also include a value called `desc`. The table structure here is a bit different from what was described above; this was chosen to have an easier time controlling the order of fields that are sent to the declaration.

We will create docstrings alongside our table declarations, in the following manner:
```lua {linenos=table}
return {
	{ field = "Date_Display", type = "String", desc = "Date for displaying in timelines, may contain \"approx\" etc" },
	{ field = "Date_Sort", type = "DateTime", desc = "Invisible to the reader, but must be an exact date. Can guess if the date is approximate, put qualifying information into Date_Display" },
	{ field = "IsApproxDate", type = "Boolean", desc = "If the date is approx or not - independent from how much precision to use as the display, this will just add the text (approx.)" },
	{ field = "EarliestPossibleDate", type = "Date", desc = "Based on values of approx/display date" },
	{ field = "LatestPossibleDate", type = "Date", desc = "Based on values of approx/display date" },
	{ field = "Sentence", type = "Wikitext", desc = "Display text (English-only)" },
	{ field = "SentenceWithDate", type = "Wikitext", desc = "Display text (English-only). Prefixed by display date." },
```