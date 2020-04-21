---
title: "Sublimetext for Mediawiki"
date: 2020-04-21T07:03:23Z
draft: false
tags:
    - sublimetext
    - mediawiki
summary: The "Mediawiker" plugin makes Sublime the best IDE-like experience for editing wikis, particularly when one is primarily writing Scribunto/Lua modules. Here's how I've set up my environment.
---
How do you edit code in MediaWiki? Wikipedia has a [list of text editor support options](https://en.wikipedia.org/wiki/Help:Text_editor_support), including some tools that will mirror your browser's textarea inputs, some browser addons that can help, and the somewhat primitive suggestion to consider **copy-pasting to an external text editor**.

If you scroll about [2/3 of the way down the page](https://en.wikipedia.org/wiki/Help:Text_editor_support#SublimeText) you will see a brief mention of [Mediawiker](https://github.com/tosher/Mediawiker), a SublimeText plugin that's still, as of the publication of this article, in active development.

Before I talk about Mediawiker or SublimeText anymore, let's go over some of the reasons you should consider this a necessary tool to have in the first place.

If you just want to see my recommended list of plugins, <a href="#my-recommended-plugins">click here</a> to jump.

## (Almost) Everything Else Sucks
To me, it's an unconditionally required feature of a MediaWiki editing environment to function without the use of a browser:
* Requiring browser use for editing also requires that you use a browser to navigate to pages. If you configure your browser nicely, with a lot of aliased bookmarks for easier navigation (I'll write a dedicated post about this eventually), this can be reasonable, but that's a huge investment of its own.
* Browsers crash a lot more often than text editors do.
* Having a ton of tabs open for editing a wiki can lead to your browser crashing even more often, which is annoying for other browser usage; and to maintain a reasonable navigation with this many tabs requires again almost as much investment as just setting up Sublimetext for this purpose.
* I don't want to have to keep light-mode windows open, ever.

So a textarea-mirroring option isn't a perfect and easy solution.

Copy-pasting to a text editor is completely impractical - it can quadruple the number of actions needed to make a small change to code, tempting you to make edits in browser, which in turn vastly increases the chances of messing up. Also, it's extremely easy to lose track of which tab of one goes to which tab of the other, and end up erasing crucial production code instead of a sandbox on accident. (I got around this by making an Abuse Filter that required the name of a module to be in a comment inside of it, but that's ridiculous overhead where my "tooling" is affecting the content of the wiki, and unacceptable in some situations.)

MediaWiker isn't the only plugin across all IDEs that does what it does; in particular there's [mediawiki.el](https://github.com/hexmode/mediawiki-el) for Emacs, maintained by hexmode, but I think Sublimetext is an incredibly nice editor, and if you don't already use an editor with a plugin like this, Sublime with Mediawiker should be your choice.

## What Mediawiker Does
Once you've [set up](https://github.com/tosher/Mediawiker/wiki/Howto) Mediawiker, you can open any page on any of your configured wikis by typing a page name into an input box or using a dropdown list of a configurable number of recent pages for each wiki. You can edit, save, preview, move, delete. You can check the WhatLinksHere, highlight redlinks, show and jump through the table of contents. It also has a MediaWiki syntax highlighter (though I find this nearly useless because MediaWiki is such a hard language to provide meaningful highlighting for). All of this is possible without once interacting with the browser.

It also automatically recognizes the type of syntax highlighting that should be used based on page name - modules (namespace 828) are automatically Lua, unless the page name ends in `/doc`; pages that end in `.css` are CSS, etc.

Recently, the developer added the ability to set up a Jinja template for new pages' preloads, using logic based on page name, namespace, etc (though previously this was easy to do by slightly tweaking the extension's code). This lets you start out with all of your common imports in a Lua module, or just the text `<includeonly></includeonly><noinclude>{{documentation}}</noinclude` if you create a new template.

As an example, here's my current new-page template (note the whitespace is commented, which is kind of annoying, but, hey, we work with MediaWiki after all).
```jinja
{%- if page_namespace == 828 -%}
    {%- if (page_name|lower).endswith('/doc') -%}{#
 ----------------------------------------- 
   -------- module documentation --------- 
 ----------------------------------------- 
#}<includeonly>{{ "{{luadoc}}" }}[[Category:Lua Modules]]</includeonly>{# end
          -----------------------
          
    #}{%- elif (page_name|lower).endswith('/i18n') -%}{#
 ----------------------------------------- 
         -------- i18n module --------- 
 ----------------------------------------- 
#}return {
    ['en'] = {
        
    },
}{# end
          -----------------------
          
    #}{%- elif (page_name|lower).startswith('module:cargodeclare/') -%}{#
 ----------------------------------------- 
     -------- Cargo dec module --------- 
 ----------------------------------------- 
#}return {
    { field = "", type = "", desc = "" },
}{# end
          -----------------------
          
    #}{% else -%}{#
 ----------------------------------------- 
       -------- normal module --------- 
 ----------------------------------------- 
#}local util_args = require('Module:ArgsUtil')
local util_cargo = require("Module:CargoUtil")
local util_html = require("Module:HtmlUtil")
local util_map = require('Module:MapUtil')
local util_table = require("Module:TableUtil")
local util_text = require("Module:TextUtil")
local util_vars = require("Module:VarsUtil")
local i18n = require("Module:I18nUtil")
local lang = mw.getLanguage('en')
local h = {}
local p = {}
function p.main(frame)
    local args = util_args.merge()
    i18n.init('{{ page_name | replace('Module:', '') }}')
end
return p{# end
          -----------------------
          
    #}{% endif %}
{%- elif (page_name|lower).endswith('/js') -%}{#
 ----------------------------------------- 
          -------- js page --------- 
 ----------------------------------------- 
#}$(function() {
    
});{# end
-----------------------
          
    #}{% else -%}{#
    NOTE: default text 
#}{{ page_text }}{# end
-----------------------
          
#}{% endif %}
```

## Starting with Sublimetext
Admittedly Sublimetext has a pretty steep learning curve. After installing it, the first thing you'll want to do is [install Package Control](https://packagecontrol.io/installation). The command palette that it tells you to use is the main workflow for doing *anything* with Sublime, unless you've configured hotkeys for a particular action. Press Ctrl/Cmd+Shift+P, and then start typing. The filter allows any number of characters in between consecutive characters you type, so if you type for example `PCI`, you will get "**P**ackage **C**ontrol: **I**nstall Package". From here, you can install anything you like.

The high learning curve comes from the fact that Sublimetext has basically 4 methods of doing anything, and none of them is really an **interface**.
* The command palette lets you select commands.
* There's a console, and sometimes commands will open an input box where the console is. Find-replace is also done through this pane at the bottom of the screen.
* It's a text editor, and so you can...edit text. (I wasn't sure if this one should even count.)
* Everything else is done by editing config files. Setting hotkeys? Config file. Choosing syntax highlight scopes and colors? Config file. Preferences for a plugin? Config file. Picking a theme? Config file. There are some plugins you can add to get better approximations to actual interfaces (for example [ColorPicker](https://packagecontrol.io/packages/ColorPicker)), but that's still accessed by opening a config file and pressing a hotkey.

So it's pretty safe to say, you need to be comfortable with the idea of using config files and CLI-style inputs to be able to use this tool. It took me a good month or two to get used to where all of the files were, what goes where, how to relate an action definition in one place to a hotkey definition elsewhere, etc. I did a lot of Googling, and you should expect to do the same.

## My Recommended Plugins
Okay, this is the reason I wrote this post!

Note that I'm giving everything json syntax highlighting, just because it looks right, but Sublime settings aren't actually json - you are allowed trailing commas and comments.

Currently, my Package Control settings file says I have the following installed:
```json
{
    "installed_packages":
    [
        "1337 Color Scheme",
        "3024 Color Scheme",
        "Atomic Color Scheme",
        "AutoFoldCode",
        "AutoHotkey",
        "Boxy Theme",
        "BracketHighlighter",
        "Color Highlighter",
        "ColorPicker",
        "ColorSchemeEditor",
        "CSS3",
        "Fold Javascript Functions",
        "Hugofy",
        "ImageMagick",
        "Indent Respectful Sort",
        "Jinja2",
        "LuaAutocomplete",
        "LuaExtended",
        "MarkdownEditing",
        "MarkdownLivePreview",
        "MarkdownPreview",
        "Material Color Scheme",
        "Material Monokai",
        "Mediawiker",
        "Minteresting",
        "Package Control",
        "PackageDev",
        "PackageResourceViewer",
        "Pane Resizer",
        "Predawn",
        "RegReplace",
        "Schemr",
        "SCSS",
        "SetWindowTitle",
        "SublimeCodeIntel",
        "Super Calculator",
        "Tab Filter",
        "Text Pastry",
        "Theme - Afterglow",
        "TOML",
        "Wrap Plus"
    ]
}
```

Because half of this article was about Mediawiker, I won't discuss it further below.

### Meta
These are plugins that make configuring Sublime easier/possible:
* Package Control (obviously)
* [PackageResourceViewer](https://packagecontrol.io/packages/PackageResourceViewer) - dependency for some other plugins
* [ColorSchemeEditor](https://packagecontrol.io/packages/ColorSchemeEditor) - customize any color scheme you install (this one is extremely necessary for me)
* [Schemr](https://packagecontrol.io/packages/Schemr) - easily swap between and preview color schemes, including support for syntax-specific schemes. I like to use one color scheme per syntax type, and I highly recommend this practice as it makes switching between languages a lot easier.
* [ColorPicker](https://packagecontrol.io/packages/ColorPicker) - as mentioned earlier, QOL for making it slightly easier to choose hex values

### General
* [Tab Filter](https://packagecontrol.io/packages/Tab%20Filter) - press ctrl+T (or the hotkey of your choice) to tab between all currently-open tabs. Unfortunately is not cross-window.
* [SetWindowTitle](https://packagecontrol.io/packages/SetWindowTitle) - This lets you have actually-readable window names in your taskbar, instead of 5 different windows of Sublime that all say `Modu` and nothing else. Name your projects, and be very liberal about saving new ones; you can always purge unused workspaces later.
* [Wrap Plus](https://packagecontrol.io/packages/Wrap%20Plus) - actually I don't use this, but it might be nice if you're editing content pages rather than code.
* [Pane Resizer](https://packagecontrol.io/packages/Pane%20Resizer) - provide hotkeys to increase/decrease size of current pane, nice when you have two-column view (which you should always use)

### Color Schemes
I'm not going to link all of these, just browse themes and install whatever you like.
* 1337 Color Scheme
* 3024 Color Scheme
* Atomic Color Scheme
* Boxy Theme
* Minteresting
* Material Color Scheme
* Material Monokai
* Predawn
* Theme - Afterglow

### Syntax Highlighting
I'm not going to link these either, they're pretty self-explanatory.
* AutoHotkey
* CSS3
* Jinja2 - you will want this for Mediawiker's settings file
* LuaExtended - was a big improvement over built-in Lua highlighting until March 2019 and the release of Sublime 3.2; now built-in is WAY better (and I should probably delete this plugin)
* MarkdownEditing
* SCSS
* TOML

### Design
* [Color Highlighter](https://packagecontrol.io/packages/Color%20Highlighter), see the CSS colors you are using
* [ImageMagick](https://packagecontrol.io/packages/ImageMagick), dependency of Color Highlighter
* [MarkdownLivePreview](https://packagecontrol.io/packages/MarkdownLivePreview), nice for Github READMEs and this blog, but sometimes it crashes
* [MarkdownPreview](https://packagecontrol.io/packages/MarkdownPreview), not a live preview, but it doesn't crash - choose a preview command from Command Palette to open a preview in browser.

### Other
I'm putting this second-to-last instead of last since the next section is significantly longer.
* [Hugofy](https://packagecontrol.io/packages/Hugofy) - Utilities when working with a Hugo website (like this blog). Actually I'm not using it yet though because I'm still SSHing through WinSCP and opening files in Sublime from there, and I'd need to be SSHing through Sublime to use this. I've seen [this](https://codexns.io/products/sftp_for_sublime) plugin recommended [here](https://stackoverflow.com/questions/15958056/how-to-use-sublime-over-ssh), but since it's not free I want to make sure I'd actually use it before installing.
* [PackageDev](https://packagecontrol.io/packages/PackageDev) - this one is exciting! It makes it easy to write short snippets of Python code without going through the hassle of actually creating a full plugin. Maybe it should have been in the `Meta` section, since you can also use it for customizing color schemes, syntax definitions, etc. In particular I use it for [prev_next_view_in_group_command](https://gist.github.com/vitaLee/3009871), which solved an incredibly annoying issue for me, where `ctrl+]` would continue to the opposite group instead of wrapping.

I also wrote the following, which is extremely inelegant because I never learned any of the Sublime api but it's useful to me when I have Lua tables originally as a list of 2 or 3 things inline, and then I want to expand them to have one item per row:

```python
import sublime
import sublime_plugin
import re


# adapted from https://stackoverflow.com/questions/15838945/shortcut-to-change-a-line-of-words-to-a-vertical-list-in-sublime-text-2

class SplitLineCommand(sublime_plugin.TextCommand):
    def run(self, edit, sep=",", repl= "\n", intro="{", outro="}"):
        view = self.view
        find = re.escape(sep + ' ') + '*(?! *$| *\n)'
        intro_repl = intro + repl
        intro = intro + ' *'
        outro_repl_start = sep + repl
        outro_repl_end = outro
        outro = ',? *' + outro
        repl = sep + repl
        cursors = view.sel()
        if len(cursors) == 1:
            cursor = cursors[0]
            begin_offset = 0
            end_offset = 0
            if cursor.empty():
                region = view.line(cursor)
                content = view.substr(region)
                line_str = view.substr(view.line(view.sel()[0]))
                tabs = len(line_str) - len(line_str.lstrip())
                intro_repl = intro_repl + '\t' * (tabs + 1)
                repl = repl + '\t' * (tabs + 1)
                outro_repl = outro_repl_start + ('\t' * tabs) + outro_repl_end
                content = re.sub(outro, outro_repl, content)
                content = re.sub(find, repl, content)
                content = re.sub(intro, intro_repl, content)
                view.replace(edit, region, content)
            cursors.clear()
            cursors.add(sublime.Region(region.begin() + begin_offset, region.begin() + len(content) + end_offset))
            view.run_command("split_selection_into_lines")
```
One day I will make it able to select in between braces or brackets, and require that the context of each comma it splits at not be a string, and make it support either tab or space indentation, and when I do that I'll make it a real plugin, but for now PackageDev is great for being able to throw this together in 30 minutes a few months ago, and have a useful (even if rather limited) tool.

### IDE-Like Support
* [BracketHighlighter](https://packagecontrol.io/packages/BracketHighlighter) is a huge improvement on built-in bracket highlighting
* [RegReplace](https://packagecontrol.io/packages/RegReplace) lets you configure preset regular expression find-replaces to execute on demand and do some code autocompletion
* [Indent Respectful Sort](https://packagecontrol.io/packages/Indent%20Respectful%20Sort) - see below
* [Fold Javascript Functions](https://packagecontrol.io/packages/Fold%20Javascript%20Functions) - I'm not actually sure if I use this
* [AutoFoldCode](https://packagecontrol.io/packages/AutoFoldCode) - or this
* [LuaAutocomplete](https://packagecontrol.io/packages/LuaAutocomplete) - I don't use this, it suggests too many things that are irrelevant to Scribunto. But it seems nice.
* [SublimeCodeIntel](https://packagecontrol.io/packages/SublimeCodeIntel) - this addon caused me [a lot of headache](https://river.me/blog/sublime-markup-code-intel/). I think I use it for some things at least.
* [Super Calculator](https://packagecontrol.io/packages/Super%20Calculator) - this is SO COOL but I don't actually use it that often (ever)
* [Text Pastry](https://packagecontrol.io/packages/Text%20Pastry) - improved support for multiple cursors etc - I rarely use this but it seems very nice for editing things like wikitables

#### Indent Respectful Sort
I use this to alphabetize my i18n files:
```lua
return {
    ['en'] = {
        LinkDisplay = 'Player',
        Team = 'Team', 
        Tournament = 'Tournament',
        ImageType = 'Image Type',
        Caption = 'Caption',
        IsProfileImageDisplay = 'Allowed in Infobox',
        SortDateDisplay = 'Sort Date',
        noSortDate = 'N/A (Will use tournament start date)',
    },
}
```
becomes:
```lua
return {
    ['en'] = {
        Caption = 'Caption',
        ImageType = 'Image Type',
        IsProfileImageDisplay = 'Allowed in Infobox',
        LinkDisplay = 'Player',
        SortDateDisplay = 'Sort Date',
        Team = 'Team', 
        Tournament = 'Tournament',
        noSortDate = 'N/A (Will use tournament start date)',
    },
}
```
(It *is* case sensitive, but I actually prefer that in this....case hahaha.)

#### RegReplace Example
Here's an example of what I use RegReplace for. The goal is, when I'm editing a module on the wiki, to type just
`require args` and have this autocompleted to an actual line of code:
```lua
local util_args = require('Module:ArgsUtil')
```
Here's how to make this possible.

In `reg_replace` User settings, to allow the rule to function:
```json
{
    "extended_back_references": true
}
```

In my `reg_replace` User rules:
```json
{
    "util_module": {
        "find": "^require (\\w)(\\w+)$",
        "replace": "local util_\\1\\2 = require(\"Module:\\c\\1\\2Util\")"
    }
}
```

In my `sublime-commands` file (this adds the command to Command Palette):
```json
{
    "caption": "RegReplace: Util Require",
    "command": "reg_replace",
    "args": {
        "replacements": [
            "util_module"
        ]
    }
}
```

In my user keymap file (this lets me press Alt+Shift+R to execute):
```json
{
    "keys": ["alt+shift+r"],
    "command": "reg_replace",
    "args": {
        "replacements": [
            "util_module"
        ]
    }
}
```

While this seems like a crazy amount of files to edit, the RegReplace plugin conveniently adds a menu to Preferences -> Package Settings linking you to all these places, and it also provides some examples.