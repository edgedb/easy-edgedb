---
tags: Datetime, Describing Types
leadImage: illustration_05.jpg
---

# Chapter 5 - Jonathan tries to leave the castle

Poor Jonathan is not having much luck. Here's what happens to him in this chapter:

> During the day, Jonathan decides to try to explore the castle but too many doors and windows are locked. He doesn't know how to get out, and wishes he could at least send Mina a letter. He pretends that there is no problem, and keeps talking to Dracula during the night. One night he sees Dracula climb out of his window and down the castle wall, like a snake. Now he is very afraid, and knows that Dracula is not human. A few days later he breaks one of the doors and finds another part of the castle. The room is very strange and he feels sleepy. When he opens his eyes, he sees three vampire women next to him. He is attracted to them and afraid of them at the same time. He wants to kiss them, but knows that he will die if he does. They come closer, and he can't move...

## std::datetime

Since Jonathan in Romania was thinking of Mina back in London, let's learn about `std::datetime` because it uses time zones. To create a datetime, you can just cast a string in ISO 8601 format with `<datetime>`. That format looks like this:

`YYYY-MM-DDTHH:MM:SSZ`

So a cast to an actual datetime looks like this:

```edgeql
db> select <datetime>'2020-12-06T22:12:10Z';
```

The `T` inside there is just a separator between date and time (in other words, `T` is where the _Time_ starts), and the `Z` at the end stands for "zero timeline". That means that it is 0 different (offset) from UTC: in other words, it _is_ UTC.

One other way to get a `datetime` is to use the `to_datetime()` function. {eql:func}`Here is its signature <docs:std::to_datetime>`, which shows that there are six ways to make a `datetime` with this function depending on how you want to make it. EdgeDB will know which one of the six you have chosen depending on what input you give it.

Let's do a quick detour before getting back to datetime. Inside the `to_datetime()` function you'll notice one unfamiliar type inside called a {eql:type}` ``decimal`` <docs:std::decimal>` type. A decimal is a float with "arbitrary precision", meaning that you can give it as many numbers after the decimal point as you want. This is because float types on computers [become imprecise after a while](https://www.youtube.com/watch?v=PZRI1IfStY0&ab_channel=Computerphile) thanks to rounding errors. This example shows it:

```edgeql-repl
edgedb> select 6.777777777777777; # Good so far
{6.777777777777777}
edgedb> select 6.7777777777777777; # Add one more digit...
{6.777777777777778} # Where did the 8 come from?!
```

If you want to avoid this, add an `n` to the end to get a `decimal` type which will be as precise as it needs to be.

```edgeql-repl
edgedb> select 6.7777777777777777n;
{6.7777777777777777n}
edgedb> select 6.7777777777777777777777777777777777777777777777777n;
{6.7777777777777777777777777777777777777777777777777n}
```

Similarly, there is a `bigint` type that also uses `n` for an arbitrary size. That's because even int64 has a limit: it's 9223372036854775807.

```edgeql-repl
edgedb> select 9223372036854775807; # Good so far...
{9223372036854775807}
edgedb> select 9223372036854775808; # But add 1 and it will fail
edgedb error: NumericOutOfRangeError: std::int64 out of range
```

Here as well you can just add an `n` and it will create a `bigint` that can accommodate any size.

```edgeql-repl
edgedb> select 9223372036854775808n;
{9223372036854775808n}
```

Now that we know all the numeric types, let's get back to the six signatures for the `to_datetime()` function:

```
std::to_datetime(s: str, fmt: optional str = {}) -> datetime
std::to_datetime(local: cal::local_datetime, zone: str) -> datetime
std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime
std::to_datetime(epochseconds: decimal) -> datetime
std::to_datetime(epochseconds: float64) -> datetime
std::to_datetime(epochseconds: int64) -> datetime
```

The easiest is probably the third if you find ISO 8601 unfamiliar or you have a bunch of separate numbers to make into a date. With this, our game could have a function that generates integers for times that then use `to_datetime()` to get a proper time stamp.

Let's imagine that it's May 12. It's a bright morning at 10:35 in Castle Dracula. The sun is up, Dracula is asleep somewhere, and Jonathan is trying to use the time during the day to escape to send Mina a letter. In Romania the time zone is 'EEST' (Eastern European Summer Time), and the year is (probably) 1893. We'll use `to_datetime()` to generate this.

```edgeql
select to_datetime(1893, 5, 12, 10, 35, 0, 'EEST');
```

And get the following output:

`{<datetime>'1893-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

We can also use this to see the duration between events. EdgeDB has a `duration` type that you can get by subtracting a datetime from another one. Let's practice by calculating the exact number of seconds between one date in Central Europe and another in Korea:

```edgeql
select to_datetime(2003, 5, 12, 8, 15, 15, 'CET') - to_datetime(2003, 5, 12, 6, 10, 0, 'KST');
```

This takes May 12 2003 8:15:15 am in Central European Time and subtracts May 12 2003 6:10 in Korean Standard Time. The result is: `{<duration>'10:05:15'}`, so 10 hours, 5 minutes, and 15 seconds.

Now let's try something similar with Jonathan as he tries to escape Castle Dracula. It's May 12 at 10:35 am in the `EEST` timezone. On the same day, Mina is in London at 6:10 am, drinking her morning tea. How many seconds passed between these two events? They are in different time zones but we don't need to calculate it ourselves; we can just specify the time zone and EdgeDB will do the rest:

```edgeql
select to_datetime(1893, 5, 12, 10, 35, 0, 'EEST') - to_datetime(1893, 5, 12, 6, 10, 0, 'UTC');
```

The answer is 1 hour and 25 minutes: `{<duration>'1:25:00'}`.

To make the query easier for us to read, we can also use the `with` keyword to create variables. We can then use the variables in `select` below. We'll make one called `jonathan_wants_to_escape` and another called `mina_has_tea`, and subtract one from another to get a `duration`. With variable names it is now a lot clearer what we are trying to do:

```edgeql
with
  jonathan_wants_to_escape := to_datetime(1893, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := to_datetime(1893, 5, 12, 6, 10, 0, 'UTC'),
select jonathan_wants_to_escape - mina_has_tea;
```

The output is the same: `{<duration>'1:25:00'}`. As long as we know the timezone, the `datetime` type does the work for us when we need a `duration`.

## Casting to a duration

Besides subtracting a `datetime` from another `datetime`, you can also just cast to make a `duration`. To do this, just write the number followed by the unit: `microseconds`, `milliseconds`, `seconds`, `minutes`, or `hours`. It will return a number including a number of seconds, or a more precise unit if necessary. For example, `select <duration>'2 hours';` will return `{<duration>'2:00:00'}`, and `select <duration>'2 microseconds';` will return `{<duration>'0:00:00.000002'}`.

You can include multiple units as well. For example:

```edgeql
select <duration>'6 hours 6 minutes 10 milliseconds 678999 microseconds';
```

This will return `{<duration>'6:06:00.688999'}`.

EdgeDB is pretty forgiving when it comes to inputs when casting to a `duration`, and will ignore plurals and other signs. Even this horrible input will work:

```edgeql
select <duration>'1 hours, 8 minute ** 5 second ()()()( //// 6 milliseconds' -
  <duration>'10 microsecond 7 minutes %%%%%%% 10 seconds 5 hour';
```

The result: `{<duration>'-3:59:04.99401'}`.

## Relative duration

The scene in the book today takes place on the 16th of May, 15 days after Jonathan Harker left London. Jonathan Harker was kind enough to even mark down the time of day during his first journal entry, which gives us a good idea of how much time has gone by since then. Here are the two relevant journal entries:

```
3 May. Bistritz.—Left Munich at 8:35 P. M., on 1st May, arriving at Vienna early next morning;

The Morning of 16 May.—God preserve my sanity, for to this I am reduced...All three had brilliant white teeth that shone like pearls against the ruby of their voluptuous lips. There was something about them that made me uneasy, some longing and at the same time some deadly fear.
```

Let's imagine that our `PC` named Emil Sinclair has been accomplishing some missions during this time in order to build up experience and get involved with the events in the book. It would be nice to let the player know how much time has elapsed since the game started. If the player clock currently says 8:03:17 am right now, we can calculate the duration as we did just before:

```
with
  game_start := to_datetime(1893, 5, 3, 20, 35, 0, 'UTC'),
  today := to_datetime(1893, 5, 16, 8, 3, 17, 'EEST'),
select today - game_start;
```

That gives us a duration of `{<duration>'296:28:17'}`. That's very precise, but it would be nice to show the player these units in more readable units. EdgeDB added a type called `relative_duration` in 2021 to do exactly this. A relative_duration will show up when you add or subtract local dates and local datetimes. Here is a quick example:

```edgeql
edgedb> select <cal::local_datetime>'2023-05-18T08:00:00' - <cal::local_datetime>'2023-05-16T04:06:55';
```

This gives the output `{<cal::relative_duration>'P2DT3H53M5S'}`, which has done all the calculating for us and is easy to split up into parts. Adding a few spaces makes it easy to read: `P 2D T 3H 53M 5S`. In other words:

* P - period (i.e. a period of time). This is from the ISO8601 standard.
* 2D - two days
* T - delimiter between days and time
* 3H = three hours
* 53M - 53 minutes
* 5S - 5 seconds

So let's make a `relative_duration` for our `PC`. Here we will use a function called `cal::to_local_datetime` which turns a `datetime` into a `local_datetime` by entering the `datetime` to convert and the timezone we want it to show the time of.

The player is currently in Romania so we will choose the EEST timezone. The code looks like this:

```edgeql
db> with
  game_start := to_datetime(1893, 5, 3, 20, 35, 0, 'UTC'),
  today := to_datetime(1893, 5, 16, 8, 3, 17, 'EEST'),
  select cal::to_local_datetime(today, 'EEST') - cal::to_local_datetime(game_start, 'EEST');
```

This gives us an output of `{<cal::relative_duration>'P12DT8H28M17S'}`. Perfect! Now our game can display something like `Time elapsed: 12 days, 8 hours, 28 minutes, 17 seconds` and we don't need to do any calculations to do so.

## Required links

Now we need to make a type for the three female vampires. We'll call it `MinorVampire`. These have a link to the `Vampire` type, which needs to be `required`. This is because Dracula controls them and they only exist as `MinorVampire`s because he exists.

```sdl
type MinorVampire extending Person {
  required link master -> Vampire;
}
```

Now that it's required, we can't insert a `MinorVampire` with just a name. It will give us this error: `ERROR: MissingRequiredError: missing value for required link default::MinorVampire.master`. So let's insert one and connect her to Dracula:

```edgeql
insert MinorVampire {
  name := 'Woman 1',
  master := assert_single(
    (select Vampire filter .name = 'Count Dracula')
  ),
};
```

You need to put the query for getting Count Dracula in parentheses as you know from earlier examples. Then you need to put all that inside the `assert_single()` function. This function makes sure that there's no more than a single element in the set it is given. This is necessary because EdgeDB doesn't know that there is only one 'Count Dracula' and we need to provide only one Vampire as the master (remember, `required link` is short for `required single link`). If we tried this without the `assert_single()` function we would get the following error:

```
error: possibly more than one element returned by an expression for a link 'master' declared as 'single'
```

Also note that `assert_single()` will return an error (a `CardinalityViolationError`) if more than one element is returned, so make sure to only use `assert_single()` if you are sure that there is only one element. In principle it's sort of like a "trust me, there is only one element" sort of function.

## Using the 'describe' keyword to look inside types

Our `MinorVampire` type extends `Person`, and so does `Vampire`. Types can continue to extend other types, and they can extend more than one type at the same time. The more you do this, the more annoying it can be to try to combine it all together in your mind. This is where `describe` can help, because it shows exactly what any type is made of. There are three ways to do it:

- `describe type MinorVampire` - this will give the {ref}`DDL (data definition language) <docs:ref_eql_ddl>` description of a type. DDL is a lower level language than SDL, the language we have been using. It is less convenient for schema, but is more explicit and can be useful for quick changes. We won't be learning any DDL in this course but later on you might find it useful sometimes. For example, with it you can quickly create functions without needing to do an _explicit_ migration.  And if you understand SDL it will not be hard to pick up some tricks in DDL.

(Note though the word _explicit_ there: using DDL still results in a migration, just an _implicit_ one. In other words, a migration happens without calling it a migration. It's sort of a quick and dirty way to make changes but for the most part proper migration tools with SDL schema is the preferred way to go.)

Now back to `describe type` which gives the results in DDL. Here's what our `MinorVampire` type looks like:

```
{
  'CREATE TYPE default::MinorVampire EXTENDING default::Person {
    CREATE REQUIRED LINK master -> default::Vampire;
};',
}
```

The `create` keyword shows that it's a series of quick commands, which is why the order is important. In other words, SDL is _declarative_ (it _declares_ what something will be without worrying about order), while DDL is _imperative_ (it's a series of commands to change the state). Also, because it only shows the DDL commands to create it, it doesn't show us all the `Person` links and properties that it extends. So we don't want that. The next method is:

- `describe type MinorVampire as sdl` - same thing, but in SDL.

The output is almost the same too, just the SDL version of the above. It's also not enough information for what we want now:

```
{
  'type default::MinorVampire extending default::Person {
    required link master -> default::Vampire;
};',
}
```

You'll notice that it's basically the same as our SDL schema, just a bit more verbose and detailed: `type default::MinorVampire` instead of `type MinorVampire`, and so on.

- The third method is `describe type MinorVampire as text`. This is what we want, because it shows everything inside the type, including stuff from the types that it extends. Here's the output:

```
{
  'type default::MinorVampire extending default::Person {
    required single link __type__ -> schema::Type {
        readonly := true;
    };
    optional single link lover -> default::Person;
    required single link master -> default::Vampire;
    optional multi link places_visited -> default::Place;
    required single property id -> std::uuid {
        readonly := true;
    };
    required single property name -> std::str;
};',
}
```

(Note: `as text` doesn't include constraints and annotations. To see those, add `verbose` at the end: `describe type MinorVampire as text verbose`. You'll learn about annotations in Chapter 14.)

The parts that say `readonly := true` we don't need to worry about, as they are automatically generated (and we can't touch them). For everything else, we can see that we need a `name` and a `master`, and could add a `lover` and `places_visited` for these `MinorVampire`s.

And for a _really_ long output, try typing `describe schema` or `describe module default` (with `as sdl` or `as text` if you want). You'll get an output showing the whole schema we've built so far.

So if `type` comes after `describe` for types and `module` after `describe` for modules, then what about links and all the rest? Here's the full list of keywords that can come after describe: `object`, `annotation`, `constraint`, `function`, `link`, `module`, `property`, `scalar type`, `type`. If you don't want to remember them all, just go with `object`: it will match anything inside your schema (except modules).

[Here is all our code so far up to Chapter 5.](code.md)

<!-- quiz-start -->

## Time to practice

1. What do you think `select to_datetime(3600);` will return, and why?

   Hint: check the function signatures above (or [in the docs](https://www.edgedb.com/docs/stdlib/datetime#function::std::to_datetime)) and see which one EdgeDB will pick when you enter 3600.

2. Will `select <int16>9 + 1.06n is decimal;` work? And if it does, will it return `{true}`?

3. How many seconds went by between 5:00 am on Christmas Day 2003 in Turkmenistan (TMT) and 7:00 pm on New Year's Eve for the same year in Uzbekistan (UZT)?

4. How would you write the same query using `with` for each of the two times?

5. What's the best way to describe a type if you only want to see how you wrote it?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _One of the women vampires to her sisters: "He is young and strong; there are kisses for us all..."_
