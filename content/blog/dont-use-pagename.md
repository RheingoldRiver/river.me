---
title: "Don't use _pageName"
date: 2020-07-14T14:05:06Z
draft: false
tags:
    - cargo
    - best_practices
summary: Sometimes it can seem convenient to use the built-in _pageName field that Cargo provides; however, _pageName is a metadata field, and you should take care to use it only as such.
---
Cargo provides a number of "metadata" fields in every table, including and most notably `_pageName`. Because they are defined by the software, they have absolute definitions and cannot have these definitions altered. They should be treated as metadata fields and only used in contexts where metadata, as opposed to data, is desired. If you treat them as data, you will permanently couple your database schema to the physical layout of your wiki, when these layers should be segregated as much as possible.

While I've titled this article "Don't use _pageName" for dramatic effect, perhaps a more appropriate title would be "Proper and improper usages of _pageName."
## Cargo's metadata fields
When you create a new Cargo table, you get a couple fields for free:
* `_pageName` holds the prefixed (includes namespace) name of the page the data is stored on.
* `_pageTitle` holds the unprefixed (doesn't include namespace) name of the page the data is stored on.
* `_pageID` holds the MediaWiki ID of the page the data is stored on.
* `_pageNamespace` holds the namespace number of the page the data is stored on.
* `_ID` is an arbitrary numerical ID assigned to the row being stored.

See more details on the [official Cargo docs page](https://www.mediawiki.org/wiki/Extension:Cargo/Storing_data#Database_storage_details).

## Valid use cases
Before I talk about when and why you shouldn't use these fields, let's talk about cases where they're in fact helpful:
### Debugging & error report generation
By far the broadest use case for these metadata fields is debugging, especially when throwing together ad-hoc queries. A very quick way to look for errors:
```
{{#cargo_query:tables=TableName
|where= ValueThatShouldNotBeNull IS NULL
|fields= _pageName
}}
```
Something like this can be extremely convenient to run and generate error reports.

### Providing edit links
If you are aggregating results from multiple pages, editors will need help locating the input data when they see a query, and you should provide them with as much help as possible to do so. Naturally, using `_pageName` here makes sense - after all, the data is guaranteed to be stored from this page - and in fact, the arguments later on about when *not* to use `_pageName` apply in reverse here; you would couple your data abstractions to your wiki structure by **not** using `_pageName` in this case.

The common theme in this example and the previous one is that **we actually want metadata, not data, in both cases**.

### Creating metadata reports about your wiki
And of course, maybe you literally just want to generate metadata reports. One example is to ensure that you are never storing more than 100 rows to a particular table on any page; in this case it's completely appropriate to both group by and query the `_pageName` field.

### Joining a parent table to a list-type child (not _pageName)
Unlike most of this article, this case refers to the `_ID` field, rather than `_pageName`. Normally I would say you should avoid using `_ID` since it's an automatically-generated, non-static, arbitrary field with no inherent meaning. However, the syntactical sugar `HOLDS` relies on this field, and it's also okay for you to use it when you're explicitly writing out the `HOLDS` join.

Why? (Feel free to skip this part if you want to just trust me.) Because the list-type field is guaranteed to permanently be a child of its parent, any time the table is rebuilt or `_ID` is changed for another reason, the software guarantees that your `_rowID` in the child is identical to the `_ID` in the parent; in other words, without any code or cache updates required, at any point in time, your queries will be based on internally-consistent and correct data, no matter how you redesign your schema or rearrange your data.

## _pageName as entity name - don't do this
The most common usage of `_pageName` is also an incorrect use of it: When an entity name matches its page name in all cases, using `_pageName` as your Entity key is very tempting - but it should never be done.

Let's say your page is called "New York City" and is a page about, surprise, New York City. You might be tempted to think, "ah, I shouldn't duplicate the city name in this table! I will just use `_pageName`!" On the surface, you're right. In fact your table might look something like this:

```
{{#cargo_store:_table=Cities
|City={{PAGENAME}}
<!-- snip -->
}}
```
Yikes, we're *literally* duplicating the `_pageName` field. Surely we can do better! Actually, we can't, and this setup is correct.

(n.b. `_pageName` actually corresponds to `{{FULLPAGENAME}}` whereas `{{PAGENAME}}` is `_pageTitle`, but these are the commonly-used fields/values I see used.)

I'm going to repeat a sentence from the introduction to this post:

> If you treat [`_pageData`] as data [as opposed to metadata], you will permanently couple your database schema to the physical layout of your wiki, when these layers should be segregated as much as possible.

### Semantic value of field names
We should also recognize that there's semantic value from naming a field accurately. If our field contains the name of a city, it should be called `City` or `CityName`, not `_pageName`.

I also want to bring up the relational model of data - technically, the relational predicate of the `_pageName` field is something along the lines of "is stored on page X on this wiki" - not a very pertinent piece of information. Especially since Cargo doesn't natively support docstrings (though [I get around that to some extent](https://river.me/blog/cargo-docstrings/)), it feels somewhat futile to insist that your `_pageName` field is actually an entity name and this part of the predicate is really saying `New York City is a city that...`.

Another point worth mentioning here is standardization - especially if people use your API for queries, in which case an alias of the `_pageName` field is required, since it starts with an underscore, you will have a lack of standardization in aliases used in logic following queries (e.g. `_pageName=City` vs `_pageName=CityName` vs `_pageName=Name` vs `_pageName=Page`, etc, all referring to the same field - perhaps even to the same field in the same query, just in different applications!). This will make code harder to copy-paste and adapt and/or learn from, and it will also make insource code searches less likely to yield useful results. Providing clear and concise field labels for all your fields, especially your primary key, is extremely important for standardization.

### Moving stores to aggregate locations
I'm of the opinion that in the majority of cases, decentralized data with centralized queries is the proper approach to adding Cargo to a wiki, but it's possible to do the inverse, and have centralized data which is then queried from individual pages. This still obeys a single source of truth, and if for example you have an external application generating structured data, it can be easier to copy-paste the contents of one single file rather than have an involved script to update many pages at once.

If you start out with a small wiki with all data entered manually, maybe you go for the decentralized input paradigm. In this case all your `_pageName` fields will line up with the right entity name. But if later on, you restructure, because you have enough buy-in from your organization you need to automate data entry, but not enough buy-in to justify writing a bunch of code - switching to centralized data can satisfy both of these requirements, and suddenly `_pageName` is meaningless because every row of your table has the exact same page name.

### Creating "data" pages
In somewhat of an inverse situation to the above, you may encounter a situation where you have too much data on a page to store it all at once without the page being prohibitively slow. Let's imagine a music wiki where you want to store a table of `Songs`. Songs don't get their own pages; instead they live on artists' pages. Demographic information about the artist goes in `Artists`, so you want to join on the name of the artist (which is the same as `_pageName`). But some artists have over 400 songs, and you can't store those all on the overview page. So instead, you create a `SongsData` template that just stores information like `SongTitle`, `AlbumTitle`, and `Duration`, and put that on a page called `Data:ArtistName` (maybe you also have `Data:ArtistName/2` and `Data:ArtistName/3`). Then you query this on the overview page. Again, single source of truth is preserved, but `_pageName` is useless, and you have to use `ArtistName` as the key to join on instead.

### Splitting content to subpages
Maybe today you are storing `EmployeeAssignments` on the employee pages. So it might seem natural to join EmployeeAssignments to Employees on `_pageName` - but in a year, you have significantly more information on your wiki, and each Employee now has three pages: Employee, Employee/Assignments, and Employee/Skills. You are storing general information about each Employee from their infobox on the main page, and then storing to EmployeeAssignments and EmployeeSkills on their respective subpages. Suddenly a `_pageName` join does nothing - you actually want to be joining on the base page! But to change it, now you have to track down every single query that used this join, and update it to join on an `EmployeeName` field instead. Ah, if only you had done this from the start!

### Disambiguations
Sometimes you want the disambiguated page name in a join - on Leaguepedia this is extremely common, in fact, because players can have the same gamertag, and so our identifier of a player will be e.g. [Wolf (Lee Jae-wan)](https://lol.gamepedia.com/Wolf_\(Lee_Jae-wan\)) instead of `Wolf`, and we'll write `Wolf (Lee Jae-wan)` every time we mean this particular player. But maybe you have Chicago (Office) and Chicago (City) pages on your wiki, and whenever you refer to the office you just write Chicago, and you don't actually want the fully-disambiguated name as part of the join. Then `_pageName` cannot be the key of your `Offices` table (or your `Cities` table!).

Or suppose the converse holds. Let's go back to Leaguepedia: Say in our `Players` table, we want to use as our primary key Faker (Lee Sang-hyeok) even though the page is just called [Faker](https://lol.gamepedia.com/Faker), to avoid any potential disruption in the event of a future disambiguation. In this case using `_pageName` would prohibit us from doing so - and if we made the decision to pre-disambiguate a year into the project, but had started out using `_pageName`, we'd be facing a mountain of technical debt to change our minds, whereas changing the value of `Player` a year into the project would be no big deal at all. (In reality, this is unlikely to happen for two reasons: first, this is not a unique key; there have been cases of players with both matching gamertags and "IRL" names. And second, there are a lot of players whose real names we have no record of!)

### _pageName as foreign key - especially bad
For convenience I assumed in all of the above examples that your `_pageName` was your **primary** key. But of course it could also be a **foreign** key. Suppose, for example, you are looking at the `StreetArt` table, stored to on various city pages. You could consider using `_pageName` in place of a dedicated field called `City`, but you will run into all of the problems listed above - and even more, because in this case, it's not even less clear what "inherent" semantic meaning `_pageName` holds (if any)! Certainly do not do this!

## Doing the right thing is really easy
Let's revisit the solution to this `_pageName`-as-data-not-metadata problem:

```
{{#cargo_store:_table=Cities
|City={{PAGENAME}}
<!-- snip -->
}}
```
We write literally *one line* and all our problems are solved? Yep! All we need to do is provide a field that we are in control of, so that if we ever need to alter this in the future, we don't have to adjust any of our queries or joins. And if it's *not* this easy, then `_pageName` is **already** invalid for our current use case!

## Conclusion
Respecting the semantic meaning of Cargo's metadata fields as actually being metadata is both appropriate and important. Even if Cargo itself doesn't refer to its automatically-populated fields of `_pageName`, `_pageTitle`, etc., as metadata, these fields are absolutely metadata and not data. Treat them accordingly, and don't be afraid to create in-practice-identical but semantically different fields.