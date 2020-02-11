---
title: "Fixing autocomplete in Sublimetext Markdown syntax"
date: 2020-02-11T05:05:25Z
draft: false
summary: How to disable autocompletion in SublimeText markdown syntax
tags:
    - sublimetext
---

This post is an attempt to make the solution to a problem I spent over two hours solving be more visible. It's possible that copying the json code snippet below into your Sublime settings for the plugin `SublimeCodeIntel` will fix your issue; if so, great! If not, here's a bunch of context that should hopefully help you debug your issue.

**The Problem:** When using a Markdown syntax highlighter (such as MarkdownEditing), undesired autocomplete is shown when typing text. (This came up because I was starting to write content for this blog.)

**The Normal Solution:** Typically, you would fix this problem by setting `auto_complete: false` in your syntax-specific settings. Maybe this will fix your issue.

**The Problem With The Normal Solution:** There are some extensions that have their own autocomplete functionality that works separately from Sublime's built-in `auto_complete`. The one that I have is `SublimeCodeIntel`, and while I was searching for issues I also saw mention of `Jedi` and `CodeComplice`. If you're trying to check if you have such a package, you can check the [auto-complete label on Package Control](https://packagecontrol.io/browse/labels/auto-complete). To fix this unwanted autocomplete, you need to disable your extension's own autocomplete.

**The Problem With The Problem:** There are two *separate* syntax highlights, (probably) neither of which you want.
1. Markdown's own autocomplete - this will give you a suggestion of any word in the entire open document.
2. Markdown autocomplete is *also* showing you HTML autocomplete, inside of Markdown syntax. This makes sense in general, and also if you check current scope (default keybind is ctrl+alt+shift+P) you can tell the scope is considered HTML, specifically `text.html.markdown` or `text.html.markdown.gfm` or something along these lines.

**The Actual Solution:** So, option 1 is just disable whatever autocompletion you have. But let's say you want to keep it. In SublimeCodeIntel, you will need to add the following:

```json
    "codeintel_live_disabled_languages":
    [
        "markup",
        "markdown",
        "html",
    ],
    "codeintel_enabled_languages":
    [
       "JavaScript", "SCSS", "Python",
       "Ruby", "Python3", "XML", "Sass", "HTML5", "Perl", "CSS",
       "Twig", "Less", "Node.js", "TemplateToolkit", "PHP"
    ],
```
The languages you have may vary, the important part is to disable in markup AND markdown, and to REMOVE html from the enabled languages. The enabled languages setting overwrites the disabled languages list.

Note that you may actually want to keep the HTML autocomplete, since those suggestions are more helpful than all of the markdown/up suggestions which are "literally any word on the entire page."

**Bonus Information:** Related to this, MarkdownEditing stores its settings in a *really weird place*. Sublime has a *lot* of settings files, and they work typically like this: Global default is overridden by global user, which is overridden by language default, which is overridden by language user. Typically you get to language settings by Preferences -> Settings - Syntax Specific.

So you would expect to find and override Markdown settings at the Syntax Specific file for Markdown right? Wrong.

MarkdownEditing saves in *package-language* files, which are available by using the Preferences -> Package Settings -> Markdown Editing menus. And since it adds three separate syntaxes (Markdown, Markdown GFM, and MultiMarkdown), the package settings are split into three separate per-syntax files. So, if you are attempting to update your MarkdownEditing settings, make sure you use this menu path instead of the normal syntax-preference workflow.