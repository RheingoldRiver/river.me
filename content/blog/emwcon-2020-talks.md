---
title: "My EMWCon 2020 Talks"
date: 2020-04-07T21:43:52Z
draft: false
tags:
    - mediawiki
summary: At EMWCon 2020, I gave one talk and hosted two Create Camp sessions, discussing different aspects of usage of Cargo and Lua in MediaWiki.
description: At EMWCon 2020, I gave one talk and hosted two Create Camp sessions, discussing different aspects of usage of Cargo and Lua in MediaWiki.
---
[EMWCon Spring 2020](https://www.mediawiki.org/wiki/EMWCon_Spring_2020) was supposed to be held in Sandusky, Ohio, but was moved online due to Coronavirus. I participated from home and gave one talk as well as two Create Camp sessions. Unsurprisingly, they all discussed usage of Cargo and Lua with MediaWiki in some capacity.

## Cargo/Lua Best Practices, and When Not To Use Them
This talk was a bit ambitious in scope and could have lasted significantly longer than the 30 minutes I had; I showcased some new infrastructure on Leaguepedia, gave a quick overview of what I'm calling the "0th normal form" (and will eventually write a full blog post about), and discussed some best practices in Lua when working with Cargo, including a significant amount of data casting in a wrapper of any call to `mw.ext.cargo.query`.

* [Video recording](https://www.youtube.com/watch?v=iy89cJIH4Sc)
* [Slides](https://docs.google.com/presentation/d/1ssyNqDLsVLQmPMGyUmEjXAyvksdPzKCKxul89_prLtE/edit#slide=id.p)

## Create Camp: Data Normalization in Cargo
In this talk, I went more indepth into my discussion of appropriate data normalization in MediaWiki, including cases when it's not appropriate to normalize due to content scoping constraints. After introducing the idea of primary keys (which don't exist in Cargo in any user-facing capacity) and the first three Normal Forms, I justified my definition of the 0th normal form, explained why it isn't a rigorous term, and gave an example of a case when my architectural decision was modified because of this principle.

* [Video recording](https://www.youtube.com/watch?v=jGL2u7F-1ZQ)

## Create Camp: An Introduction to Lua
A repeat of a session I led at EMWCon 2019 in person, this workshop was an introduction to using Lua via the Scribunto extension in MediaWiki. I described reasons for using Lua and coded a nontrivial but very simple example to create a flexible table displaying data from Cargo with a one-to-many relation. After creating one format of the data, I described a potential requirements change and demonstrated how the display could be changed easily as needed.

* [Video recording](https://www.youtube.com/watch?v=-YbvRj7Bs-4)
* [Module written](https://esports.gamepedia.com/Module:CatsAdopted2)