---
leadImage: illustration_00.jpg
---

# Chapter 0 - EdgeDB and the book Easy EdgeDB

Thanks for checking out our new book to learn Easy EdgeDB!

We've had quite the journey so far up to the release of this book. It began in spring 2019 with the public release of [EdgeDB Alpha 1](https://www.edgedb.com/blog/edgedb-1-0-alpha-1), showing for the first time the possibilities of an open source database built on top of PostgreSQL that combines the simplicity of a NoSQL database with relational model’s powerful querying, strictness, consistency, and performance. To see what's going on since then, see [our blog](https://www.edgedb.com/blog) where we announce the latest developments in the EdgeDB ecosystem.

Much of our learning material can be found in the [interactive tutorial](https://edgedb.com/tutorial), [documentation](https://www.edgedb.com/docs/) and [blog posts](https://www.edgedb.com/blog/we-can-do-better-than-sql), but we also wanted to put something together that immerses the learner for longer - a full book. The result is Easy EdgeDB: a book large enough (and fun enough) to make any interested beginner into a comfortable intermediate EdgeDB user by the end.

- [A new JavaScript driver](https://github.com/edgedb/edgedb-js) in Alpha 2.0 and more powerful bindings in Alpha 4.0,
- [We took our cli and Rewrote It In Rust](https://github.com/edgedb/edgedb-cli)! Unsurprisingly, the performance improvements were substantial.
- [Constraints were added to entire types](https://www.edgedb.com/blog/edgedb-1-0-alpha-5-luhman) and JSON casting became as easy as typing `<json>` before a query (Alpha 5.0),
- Native {ref}`GraphQL support <docs:ref_graphql_index>` has been improved including multiple mutations in a single query, and an `exists` filter (Alpha 5.0),
- [Improved UPDATE syntax](https://www.edgedb.com/blog/edgedb-1-0-alpha-6-wolf-359), enums, easier function syntax and cli improvement (Alpha 6.0).

Now that we've reached beta 1.0, we're even more excited to see people give it a try. Up to now our learning and introductory material has been mostly found in the [interactive tutorial](https://edgedb.com/tutorial), [documentation](https://www.edgedb.com/docs/) and [blog posts](https://www.edgedb.com/blog/we-can-do-better-than-sql). This time we've put something much longer together: a book large enough (and fun enough) to make any interested beginner into a comfortable intermediate EdgeDB user by the end.

Easy EdgeDB is a bit of a unique book. Here's how it differs from what you might expect from text of its type:

## A story to follow

The book is divided into 20 chapters in which we imagine that we are creating a database for a game. The setting for our imaginary game is the scenes and locations from the book Dracula by Bram Stoker, published in 1897. Our job is to think a little about how to represent items like characters, events, locations, and dates in the book in a database used by a role-playing game built separately. In other words, something like {ref}`Python <docs:edgedb-python-intro>`/{ref}`JavaScript <docs:edgedb-js-intro>`/[Go](https://github.com/edgedb/edgedb-go) on the frontend, and EdgeDB as the game's database. (Why Python, JavaScript and Go? Because EdgeDB has drivers for all three of those languages.)

Bram Stoker's Dracula was the perfect choice for this textbook for a few reasons:

- It's a fun read. We're excited to see you give EdgeDB a try and hope to make the introduction as painless and immersive as possible. We think that anyone who has spent an hour or so with EdgeDB will walk away excited about its possibilities, but for that to happen it's our job to make the experience a pleasant one. Bram Stoker's novel has really helped out here.
- It's a so-called _epistolary novel_: a novel written through the letters, diaries, and communications written by the main characters. That means that each entry has a date and an author, which makes it a great fit for a database.
- It's copyright free. Our book only scratches the surface of what a real database for a game based on this book would be like, but who knows? Maybe some readers will like it enough to take the concept a bit farther and make it into the real thing. And if that's the case, then an open source database software based on a book completely free of copyright is the best way to start.

## Plain English

We want to give as many people as possible the opportunity to sit down and give EdgeDB a try, and so we've opted for a style of writing that's simple and straightforward. Not baby talk, just plain English. We have three types of people in mind here:

- People unfamiliar with how to build a database but ready to understand how they work if the concepts are explained in a straightforward way,
- People who _are_ familiar with databases but maybe aren't in the mood to wade through dense verbiage for a product they've never seen before,
- People with English as a second (or third, fourth...) language who would much prefer reading a book in plain English for the same reason.

## Interactivity and experimentation

Because the book uses the events in the book for the background, we need a database to tie everything together. It will need to show the connections between the characters, locations, dates, and more. It starts with a simple schema (structure) and builds up from there, changing it as we go. The idea is to simulate the mental process for someone new to EdgeDB that has been given the task of putting this all together. That includes sometimes modifying the schema, creating new types, deleting ones that aren't used, and all the tinkering you'd see in real life.

Going through the book, we will learn how to use queries that are more and more complex. Each chapter contains the schema and inserted data that we've built up so far, and a REPL for you to experiment with. On top of that, each chapter has a number of questions for you to solve if you feel like a small challenge.

## Beauty

We looked far and wide, and didn't see any rule that a text on database software has to be dry and image free. To give a feel for the beauty of the original work (with a steampunk-ish vibe added for good measure) we teamed up with Damian Dideńko ([didiusz on Instagram](https://www.instagram.com/didiusz/)), an illustrator of 10 years from Katowice, Poland, to put together some beautiful sketches that combine the atmosphere of the book Dracula with the most important schema and query concepts per chapter. You'll soon become familiar with his illustrations but here is how he describes them and what inspires him:

> I try to take inspiration from everything that I have contact with. In my works, I like to build slightly surreal, understated/untold stories that leave the viewer room for their own interpretation. The works themselves are a loose stream of thoughts that make sense while creating, sometimes at the very end and sometimes not at all - because not everything has to make sense. My works often start with a small idea that grows into a much larger composition. I like to create works that are rich in detail, in which I sometimes hide what inspires me. They are a bit like little easter eggs for the watchful observer.

We're pleased to have teamed up with Damian to put the final touch on a book that blends the old and the new in a form that we hope will keep you turning the page as you familiarize yourself with EdgeDB and discover what it has to offer you.

[So let's get started - on to Chapter 1!](../chapter1/index.md)
