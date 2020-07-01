---
title: "Vscode 2"
date: 2020-07-01T20:15:29Z
draft: false
tags:
    - vscode
summary: In which I report a bug and discover I have to write an extension, sigh
---
See also: [VSCode 1](https://river.me/blog/vscode-1/)
## Ctrl+left arrow
My friend: Are you using VSCode for markdown?
Me: Yeah, but there's this really annoying behavior where Ctrl+Left arrow doesn't behave the same as it does in Sublime, and it's literally ruining the entire editor for me....
Him: .......
Me: Yeah, here's how you repro it....


And now that I've written repro steps, I can ignore this no longer.

* [vscode change ctrl left arrow behavior](https://www.google.com/search?q=vscode%20change%20ctrl%20left%20arrow%20behavior)
* [vscode change ctrl left when no character to the right](https://www.google.com/search?ei=sOH4Xo2VMdLXtAayqaygAw&q=vscode+change+ctrl+left+when+no+character+to+the+right&oq=vscode+change+ctrl+left+when+no+character+to+the+right&gs_lcp=CgZwc3ktYWIQAzIFCCEQqwIyBQghEKsCMgUIIRCrAjoECAAQRzoFCCEQoAE6BwghEAoQoAFQyLkMWL7TDGCG1QxoAHACeAGAAfMBiAHrJ5IBBjAuMjYuNJgBAKABAaoBB2d3cy13aXo&sclient=psy-ab&ved=0ahUKEwiNrZCOkqXqAhXSK80KHbIUCzQQ4dUDCAs&uact=5)
* [cursorWordStartLeftSelect vscode](https://www.google.com/search?q=cursorWordStartLeftSelect%20vscode)
* [vscode cursorWordStartLeft when nothing to the right of the cursor](https://www.google.com/search?q=vscode%20cursorWordStartLeft%20when%20nothing%20to%20the%20right%20of%20the%20cursor)

And then when zero of these searches were remotely helpful, I decided this was *definitely* a bug,  so I [opened a ticket](https://github.com/microsoft/vscode/issues/101243), and it's [now been patched](https://github.com/microsoft/vscode/commit/7923112cdd20575406ec547b55a99c289dd7d7e2)!

## Language configuration

Unfortunately, there's this one part of the bug report form:

```
<!-- Launch with `code --disable-extensions` to check. -->
Does this issue occur when all extensions are disabled?: Yes/No
```

This is entirely reasonable, and I wanted to do my due diligence and check (though in the end I forgot to update my ticket to answer this part!). Unfortunately, there was an update pending when I restarted VSCode.

There was a problem with the update.

Remember how I [modified a core language definition file for Markdown?](https://river.me/beta/blog/vscode-1/) Yeah, turns out you [aren't supposed to do that](https://old.reddit.com/r/vscode/comments/5nyyc1/how_do_i_change_the_language_configuration_json/).

I ended up having to reinstall VSCode completely (and restart my computer! (I REALLY hate doing this)), and in the process not only did it reinitialize my `language-configuration.json` but it even deleted my entire `.git` folder from the directory! At least I had [put it online](https://github.com/RheingoldRiver/vscode-markdown-syntax/tree/my-changes).....

So, I guess I'm (eventually) going to have to wade through docs and actually write an extension....

For now at least though, since this is just a Markdown editor, I'll just go ahead and change back to my modified file, and then know to undo and redo changes each version update. A pain, but I don't feel like learning the overhead of writing an extension just yet. For a MARKDOWN EDITOR!!!! 