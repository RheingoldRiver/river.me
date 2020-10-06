---
title: "Serving Flask from a non-root location with Nginx"
date: 2020-07-07T13:38:13Z
draft: false
tags:
    - flask
    - nginx
    - linux
summary: I couldn't find much documentation on how to serve Flask with uWSGI and Nginx from a non-root location, so, here's some.
description: I couldn't find much documentation on how to serve Flask with uWSGI and Nginx from a non-root location, so, here's some.
---

[Lack of documentation on how to serve Flask under a non-root path](https://github.com/pallets/flask/issues/1714). That's pretty much the story of why this past weekend sucked, and why I'm writing this blog post. That issue was opened in 2016, and as far as I can tell, there's still not great documentation about it. If you found this blog post in a search engine and are just looking for a solution, hopefully this helps:

```
		location /utils/parser/ {
			rewrite ^/utils/parser(/.*)$ $1 break;
			include uwsgi_params;
			uwsgi_pass unix:/home/river/flask/sbparser/sbparser.sock;
		}
```
That first line in your `nginx.conf` will "trick" Flask into thinking it is in fact being served from a root location and solve all your problems, except for, it won't fix how `url_for()` doesn't work (if that's what you're trying to fix, sorry, I gave up). From my searches I think it's maybe not possible to make `url_for()` work without Gunicorn or some other additional tool, which isn't part of my stack currently, and it doesn't seem worth it to modify something that's working when I can just use relative paths in my templates for loading scripts/styles and not care about having path hard-coded into my script; I can always `grep` later if I get *really* desperate.

Note that the non-root path issue (excluding `url_for()`, which never worked) was fine when I was running from development with `flask run` and the following nginx configuration:

```
		location /utils/parser/ {
			proxy_pass http://localhost:5000/;
		}
```

It was only when I switched to using the production setup with uWSGI that the non-root location became an issue.

Thanks to [hexmode](https://twitter.com/hexmode) and [chhopsky](https://twitter.com/chhopsky) for helping me with setup; it was chhopsky who <s>figured out the final solution</s> found the final solution on another server where he'd used this before.

[Here's the repo of my project](https://github.com/RheingoldRiver/sb_parser); I'll link some resources below and then talk about some other issues I ran into and lessons learned.
## Resources I used
Here's some resources I followed/found helpful:
* [The Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world)
    * I used part 1 (Hello World) and Part 2 (Templates).
    * Parts 3 (Web Forms) and 14 (Ajax) might have been relevant but they seemed like they were going to make me spend a lot of effort to do things "right" when I could instead do them easily with just an HTML form and a couple event handlers. Maybe I'll read more docs and do them "right" later.
    * I also didn't bother reading the deployment section because I was going to follow the Digital Ocean tutorial no matter what. Probably if I'd deployed on Heroku instead as a separate website I would have had a much easier time, but I really wanted to have all my stuff in the same place, and hey, this way I learned something right?????
* [How To Serve Flask Applications with uWSGI and Nginx on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04)
  * Technically I'm running on Debian 10 (for which there was no dedicated guide), but I don't think that made anything different
  * I didn't follow step 3 as written since I set up my app as shown in the Flask tutorial linked above
  * My nginx config also isn't identical because I was already hosting this blog as well as a couple other things (a friend of mine did the initial config months ago). Also see below for an issue I had with this.
* [What is the uwsgi_params file?](https://uwsgi-docs.readthedocs.io/en/latest/Nginx.html#what-is-the-uwsgi-params-file)
  * This is required by step 6 of the above guide but not mentioned anywhere. This may be part of the debian-ubuntu difference, or maybe it's because nginx was already installed and not by following a DO walkthrough? Not sure...but just copy that to the same directory as `nginx.conf` or whatever your config file is.
  * Found via [this SO post](https://stackoverflow.com/questions/24132137/where-should-the-uwsgi-params-file-be-located-and-what-is-its-extension)
* [rebuild uwsgi with pcre support](https://stackoverflow.com/questions/21669354/rebuild-uwsgi-with-pcre-support)
  * Not sure if this was needed but when troubleshooting why "nothing worked" this was one of the things I did, specifically I ran this inside my venv:
  
        pip uninstall uwsgi
        sudo apt-get install libpcre3 libpcre3-dev
        pip install uwsgi
* Not included: About a dozen searches & StackOverflow posts with keywords like "nginx flask non-root path" etc. None of them had anything useful. Good luck to you if you try, if you find something [let me know](https://twitter.com/RheingoldRiver)!

## Miscellaneous non-Flask issues
* One of the dependencies I needed was a library called [rapidfuzz](https://github.com/maxbachmann/rapidfuzz). I had some [issues installing it](https://github.com/maxbachmann/rapidfuzz/issues/38).
* At first I was setting up my app as `root` (I know, I know). I was getting permission errors because my `systemctl nginx` process didn't have access to the folder that my app was in. So I ended up redoing the entire installation under a new user account that I created.
* After changing user accounts, I was unable to `sudo systemctl reload sbparser` but `sudo systemctl stop sbparser; sudo systemctl start sbparser` worked fine, so I'm just using this to reload now.

## Things I learned
* It's possible to make Nginx tell pretty much any lie it wants to about where the current location it is when it forwards traffic to any application, if you try hard enough
* `systemctl` processes have user groups, and you should pay attention to these
* The line `include _` in `nginx.conf` means that you should have a file called `_` in the same directory
* How to load environment variables in Python with `python-dotenv` (ok technically not a Linux thing, but kind of?)
* How to set up basic Flask webpages and error pages

## Things I recently learned that were useful to know
I'm including this section because often I feel like when doing configs, especially in Linux, it feels like nothing that I learn compounds on itself (in extremely stark, and sad, contrast to coding). So when I find counter-examples it's worth noting that!
* You can define `EnvironmentFile` in the `[Service]` section of the `.service` file for a `systemctl` file - I did this for the first time last week for my [Discord bot](https://github.com/RheingoldRiver/river-wiki-cogs) and then did it again now for my `sbparser` process.
* Piping `journalctl` output into `less` gives it line wrapping; press `G` to jump to the end.
* Some maybe-slightly-more-than-basic `vim` navigation - `dd` to delete a line and then `p` to paste it elsewhere, `o` to insert a new line below the current one.