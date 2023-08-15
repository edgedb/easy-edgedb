---
tags: Datetime, Describing Types
leadImage: illustration_05.jpg
---

# Chapter 5 - Jonathan tries to leave the castle

Poor Jonathan is still at Castle Dracula and is not having much luck. Here's what happens to him in this chapter:

> During the day, Jonathan decides to try to explore the castle but too many doors and windows are locked. He doesn't know how to get out, and wishes he could at least send a letter to Mina **in London**. He wonders what Mina is doing right about now. **What time is it in London**, and is she okay?
>
> He pretends that there is no problem, and keeps talking to Dracula during the night. One night he sees Dracula climb out of his window and down the castle wall, like a snake. Dracula is not a human, but a monster! Now Jonathan is very afraid.
>
> A few days later he breaks one of the doors and finds another part of the castle. The room is very strange and he feels sleepy. When he opens his eyes, he sees three vampire women next to him. He is attracted to them and afraid of them at the same time. He wants to kiss them, but knows that he will die if he does. They come closer, and he can't move...

## std::datetime

Jonathan in Romania was thinking of Mina back in London, which is in a different timezone. This sounds like a good time to give the `std::datetime` type a try. To create a datetime, you can just cast a string in ISO 8601 format with `<datetime>`. The ISO 8601 format is `YYYY-MM-DDTHH:MM:SSZ`, which next to a concrete example looks like this:

```
Format:  YYYY-MM-DDTHH:MM:SSZ
Example: 2023-06-06T22:12:10Z
```

So a cast to an actual datetime looks like this:

```edgeql
select <datetime>'2023-06-06T22:12:10Z';
```

The `T` inside there is just a separator between date and time (in other words, `T` is where the _Time_ starts), and the `Z` at the end stands for "zero timeline". That means that it has no offset — no difference — from UTC: in other words, it _is_ UTC.

One way to set a timezone is to change the `T` to an offset: the hour difference between the current time zone and UTC. Our query above returns the time "22:12:10Z", so 10:12 PM in London. Let's change the `T` to `+09:00` (the timezone for Korea and Japan) and see what happens:

```edgeql
select <datetime>'2023-06-06T22:12:10+09:00';
```

The output shows us the time in UTC again. In other words, it's 1:12 pm in London when it's 10:12 pm in Korea and Japan.

```
{<datetime>'2023-06-06T13:12:10Z'}
```

One other way to get a `datetime` is to use the `to_datetime()` function. {eql:func}`Here are its signatures <docs:std::to_datetime>`, which show that there are six ways to make a `datetime` with this function depending on how you want to make it. EdgeDB will know which one of the six function signatures you have chosen depending on what input you pass in.

(Keep that point in mind, by the way! It's a hint for when we learn to write our own functions in Chapter 11.)

Let's do a quick detour before getting back to datetime. Inside one of the `to_datetime()` function signatures you might have noticed one unfamiliar type called a {eql:type}` ``decimal`` <docs:std::decimal>`. A decimal is a float with "arbitrary precision", meaning that you can give it as many numbers after the decimal point as you want. The decimal type exists because float types on computers [become imprecise after a while](https://www.youtube.com/watch?v=PZRI1IfStY0&ab_channel=Computerphile) thanks to rounding errors. This example shows the problem that floats have after too much precision:

```
db> select 6.777777777777777; # Good so far
{6.777777777777777}
db> select 6.7777777777777777; # Add one more digit...
{6.777777777777778} # Where did the 8 come from?!
```

To avoid this imprecision you can use a `<decimal>` cast, or just add an `n` to the end. This will return a `decimal` type which will be as precise as it needs to be. Now the rounding errors are gone:

```
db> select 6.7777777777777777n;
{6.7777777777777777n}
db> select 6.7777777777777777777777777777777777777777777777777n;
{6.7777777777777777777777777777777777777777777777777n}
```

Similarly, there is a `bigint` type that also uses `n` for an arbitrary size. That's because even int64 has a limit: it's 9223372036854775807.

```
db> select 9223372036854775807; # Good so far...
{9223372036854775807}
db> select 9223372036854775808; # But add 1 and it will fail
edgedb error: NumericOutOfRangeError: std::int64 out of range
```

Here as well you can just add an `n` and it will create a `bigint` that can accommodate any size.

```
db> select 9223372036854775808n;
{9223372036854775808n}
```

Now that we know all the numeric types, let's get back to the six signatures for the `to_datetime()` function:

```
std::to_datetime(s: str, fmt: optional str = {}) -> datetime
std::to_datetime(local: cal::local_datetime, zone: str) -> datetime
std::to_datetime(year: int64, month: int64, day: int64,
  hour: int64, min: int64, sec: float64, timezone: str) -> datetime
std::to_datetime(epochseconds: decimal) -> datetime
std::to_datetime(epochseconds: float64) -> datetime
std::to_datetime(epochseconds: int64) -> datetime
```

The last signature works well if you want to know the `datetime` right now and are getting the epoch seconds (the number of seconds since 1970) from your computer's system time. It's also kind of fun to just experiment with:

```
db> select to_datetime(0);
{<datetime>'1970-01-01T00:00:00Z'}
db> select to_datetime(100);
{<datetime>'1970-01-01T00:01:40Z'}
db> select to_datetime(9879879870);
{<datetime>'2283-01-30T11:04:30Z'}
db> select to_datetime(87623);
{<datetime>'1970-01-02T00:20:23Z'}
db> select to_datetime(98723987213);
{<datetime>'5098-06-09T17:46:53Z'}
```

But the easiest signature is probably the third if you find ISO 8601 unfamiliar or you have a bunch of separate numbers to make into a date. Let's give that signature a try.

Let's imagine that it's May 12. It's a bright morning at 10:35 in Castle Dracula. The sun is up, Dracula is asleep in a coffin somewhere, and Jonathan is trying to use the time during the day to find a way to send Mina a letter. In Romania the timezone is 'EEST' (Eastern European Summer Time), and the year is (probably) 1893. We'll use `to_datetime()` to generate this.

```edgeql
select to_datetime(1893, 5, 12, 10, 35, 0, 'EEST');
```

And get the following output:

`{<datetime>'1893-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

We can also use this to see the duration between events. EdgeDB has a `duration` type that you can get by subtracting a datetime from another one. Let's practice by calculating the exact number of seconds between one date in Central Europe and another in Korea:

```edgeql
select to_datetime(2003, 5, 12, 8, 15, 15, 'CET')
     - to_datetime(2003, 5, 12, 6, 10, 0,  'KST');
```

This takes May 12 2003 8:15:15 am in Central European Time and subtracts May 12 2003 6:10 in Korean Standard Time. The result is: `{<duration>'10:05:15'}`, so 10 hours, 5 minutes, and 15 seconds.

Now let's try something similar with Jonathan as he tries to escape Castle Dracula. It's May 12 at 10:35 am in the `EEST` timezone. On the same day, Mina was in London at 6:10 am, drinking her morning tea. How many seconds passed between these two events? They are in different timezones, which makes the calculation a bit annoying. Fortunately, we don't need to calculate it ourselves! We can just specify the timezone and EdgeDB will do the rest.

```edgeql
select to_datetime(1893, 5, 12, 10, 35, 0, 'EEST')
     - to_datetime(1893, 5, 12, 6,  10, 0, 'UTC');
```

The answer is 1 hour and 25 minutes: `{<duration>'1:25:00'}`.

To make the query easier for us to read, we can also use the `with` keyword to create variables. We can then use the variables in `select` below. We'll make one called `jonathan_wants_to_escape` and another called `mina_has_tea`, and subtract one from another to get a `duration`. With variable names it is now a lot clearer what we are trying to do:

```edgeql
with
  jonathan_wants_to_escape := 
    to_datetime(1893, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := 
    to_datetime(1893, 5, 12, 6,  10, 0, 'UTC'),
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

EdgeDB is pretty forgiving when it comes to inputs when casting to a `duration`. It and will ignore plurals, will recognize abbreviations, and so on:

```
edgedb> select <duration>'2 milliseconds';
{<duration>'0:00:00.002'}
edgedb> select <duration>'2 hour';
{<duration>'2:00:00'}
edgedb> select <duration>'1 seconds';
{<duration>'0:00:01'}
edgedb> select <duration>'1 H';
{<duration>'1:00:00'}
```

It even ignores symbols and irrelevant characters so even this horrible input will work:

```edgeql
select <duration>'1 hours, 8 minute ** 5 second ()()()( //// 6 milliseconds'
     - <duration>'10 microsecond 7 minutes %%%%%%% 10 seconds 5 hour';
```

The result: `{<duration>'-3:59:04.99401'}`.

It won't just accept anything, however, so there is a limit:

```
edgedb> select <duration>'three howers';
edgedb error: InvalidValueError: invalid input syntax for type std::duration: 
"three howers"
```

## Relative duration

The scene in the book today takes place on the 16th of May, 15 days after Jonathan Harker left London. Jonathan Harker was kind enough to even mark down the time of day during his first journal entry, which gives us a good idea of how much time has gone by since then. Here are the two relevant journal entries:

```
3 May. Bistritz.—Left Munich at 8:35 P. M., on 1st May, arriving at 
Vienna early next morning;
```

```
The Morning of 16 May.—God preserve my sanity, for to this I am reduced...
All three had brilliant white teeth that shone like pearls against the 
ruby of their voluptuous lips. There was something about them that made
me uneasy, some longing and at the same time some deadly fear.
```

Let's imagine that our `PC` named Emil Sinclair has been accomplishing some missions during this time in order to build up experience and get involved with the events in the book. It would be nice to let the player know how much time has elapsed since the game started. If the player clock currently says 8:03:17 am right now, we can calculate the duration as we did just before:

```edgeql
with
  game_start := to_datetime(1893, 5, 3, 20, 35, 0, 'UTC'),
  today      := to_datetime(1893, 5, 16, 8, 3, 17, 'EEST'),
select today - game_start;
```

That gives us a duration of `{<duration>'296:28:17'}`. That's very precise, but it would be nice to show the player these units in more readable units. EdgeDB added a type called `relative_duration` in 2021 to do exactly this. A relative_duration will show up when you add or subtract local dates and local datetimes. Here is a quick example:

```edgeql
select <cal::local_datetime>'2023-05-18T08:00:00'
     - <cal::local_datetime>'2023-05-16T04:06:55';
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
with
  game_start := to_datetime(1893, 5, 3, 20, 35, 0, 'UTC'),
  today      := to_datetime(1893, 5, 16, 8, 3, 17, 'EEST'),
  select
    cal::to_local_datetime(today, 'EEST') 
  - cal::to_local_datetime(game_start, 'EEST');
```

This gives us an output of `{<cal::relative_duration>'P12DT8H28M17S'}`. Perfect! Now our game can display something like `Time elapsed: 12 days, 8 hours, 28 minutes, 17 seconds` and we don't need to do any calculations to do so.

Another type called `date_duration` is useful when you only care about day to day duration. If we change the above code from `cal::to_local_datetime` to `cal::to_local_date` then we will cut off the time information and get a `date_duration` that only shows the number of days passed. So if we type this query:

```edgeql
with
  game_start := to_datetime(1893, 5, 3, 20, 35, 0, 'UTC'),
  today      := to_datetime(1893, 5, 16, 8, 3, 17, 'EEST'),
  select 
    cal::to_local_date(today, 'EEST') 
  - cal::to_local_date(game_start, 'EEST');
```

We will get the output `{<cal::date_duration>'P13D'}`. So even though only about 12 days and 8 hours have gone by, in terms of changes to the date it is 13 days. Then we could add one to it and have this output for the character playing the game:

`Day 14 of your quest...`

## Required links

The three female vampires that Jonathan encounters this chapter are controlled by Count Dracula. They have their own thoughts but Dracula can control them if he wants, and they only exist because he exists. We'll need a new type for this sort of vampire, so let's call it `MinorVampire`. These have a link to the `Vampire` type, which needs to be `required`:

```sdl
type MinorVampire extending Person {
  required master: Vampire;
}
```

Now let's do a migration to add `MinorVampire` to the schema.

With `master` as a `required` link, we can't insert a `MinorVampire` with just a name:

```edgeql
insert MinorVampire {
  name := 'Vampire Woman 1' # We never find out their names in the book
};
```

Trying this insert will give us this error: `edgedb error: MissingRequiredError: missing value for required link 'master' of object type 'default::MinorVampire'`. This is what we want. Now let's insert the same `MinorVampire` but this time we will connect her to Dracula. This link is to a single object, which means that we should filter on the name 'Count Dracula' and then use the `assert_single()` function to ensure that `master` doesn't link to more than one object.

Our three `MinorVampire` inserts look like this:

```edgeql
insert MinorVampire {
  name := 'Vampire Woman 1',
  master := assert_single((select Vampire filter .name = 'Count Dracula'))
};
insert MinorVampire {
  name := 'Vampire Woman 2',
  master := assert_single((select Vampire filter .name = 'Count Dracula'))
};
insert MinorVampire {
  name := 'Vampire Woman 3',
  master := assert_single((select Vampire filter .name = 'Count Dracula'))
};
```

Later on we will learn to add a constraint to ensure on the schema level that parameters like `name` have to be unique. And we will also learn how to insert three objects at the same time instead of doing three separate inserts like we did here.

## Using the 'describe' keyword to look inside types

Our `MinorVampire` type extends `Person`, and so does `Vampire`. Types can continue to extend other types, and they can extend more than one type at the same time. The more you do this, the harder it can be to try to picture it all together in your mind. Or you might be in the middle of using the REPL and don't want to switch to the schema file to get some information about a type. EdgeDB has a keyword called `describe` that can help in this case. There are four ways to use `describe`:

- `describe type MinorVampire` - this will give the {ref}`DDL (data definition language) <docs:ref_eql_ddl>` description of a type. DDL is the lower level language that we have seen in our migration files that end in `.edgeql` such as 00001.edgeql, 00002.edgeql, and so on. If we type `describe type MinorVampire` into the REPL, we will see the following output:

```
{
  'create type default::MinorVampire extending default::Person {
    create required link master: default::Vampire;
};',
}
```

The `create` keyword shows that DDL is a series of quick commands, which is why the order is important. In other words, SDL is _declarative_ (it _declares_ what something will be without worrying about order), while DDL is _imperative_ (it's a series of commands to change the state). Also, because it only shows the DDL commands to create it, it doesn't show us all the `Person` links and properties that it extends. So we probably don't want that. The next method is:

- `describe type MinorVampire as sdl` - same thing, but in SDL.

The output is almost the same too, just the SDL version of the above. It's also not enough information for what we want now:

```
{
  'type default::MinorVampire extending default::Person {
    required master: default::Vampire;
};',
}
```

This output is basically the same as our SDL schema, just a bit more detailed when it comes to module paths: `type default::MinorVampire` instead of `type MinorVampire`, and so on.

- The third method is `describe type MinorVampire as text`. Now the output shows almost everything we need to know inside the type, including from the types that it extends. Here's the output:

```
{
  'type default::MinorVampire extending default::Person {
    required single link __type__: schema::Type {
        readonly := true;
    };
    optional single link lover: default::Person;
    required single link master: default::Vampire;
    optional multi link places_visited: default::Place;
    required single property id: std::uuid {
        readonly := true;
    };
    required single property name: std::str;
};',
}
```

If you want a bit more information, you can add the keyword `verbose` to make the command `describe type MinorVampire as text verbose`. This will look similar to the previous output, except it also includes constraints and annotations. (We'll learn about annotations in Chapter 14.) Here is the output:

```
{
  'type default::MinorVampire extending default::Person {
    required single link __type__: schema::ObjectType {
        readonly := true;
    };
    optional single link lover: default::Person;
    required single link master: default::Vampire;
    optional multi link places_visited: default::Place;
    required single property id: std::uuid {
        readonly := true;
        constraint std::exclusive;
    };
    required single property name: std::str;
};',
}
```

The parts that say `readonly := true` are parts of each type that are automatically generated and which we can't change (hence the word `readonly`). The next part we want to scan for is `required`, giving us the minimum we need to create an object. For `MinorVampire`, we can see that we need a `name` and a `master`, and could add a `lover` and `places_visited` for these `MinorVampire`s.

The most interesting `readonly` part of an object is `__type__`, which is a link that every object type has to information about itself. We will learn more about this in Chapters 8 and 13, but if you are curious about what is inside then give it a try with the splat operator:

```
select Person.__type__ {*};
```

For a _really_ long output using `describe`, try typing `describe schema` or `describe module default` (with `as sdl` or `as text` if you want). You'll get an output showing the whole schema we've built so far.

So if we can use `describe` to learn about a `type` or our `schema`, are there other things we can describe? Indeed there are: we can describe an `object`, `annotation`, `constraint`, `function`, `link`, `module`, `property`, `scalar type`, or `type`.

If you don't want to remember them all, just go with `object`: it will match anything inside your schema (except modules). So `describe scalar type SleepState` and `describe object SleepState` will both return the same thing.

[Here is all our code so far up to Chapter 5.](code.md)

<!-- quiz-start -->

## Time to practice

1. What do you think `select to_datetime(3600);` will return, and why?

   Hint: check the function signatures above (or [in the docs](https://www.edgedb.com/docs/stdlib/datetime#function::std::to_datetime)) and see which one EdgeDB will pick when you enter 3600.

2. Will `select <int16>9 + 1.06n is decimal;` work? And if it does, will it return `{true}`?

3. How many seconds went by between 5:00 am on Christmas Day 2003 in Turkmenistan (TMT) and 7:00 pm on New Year's Eve for the same year in Uzbekistan (UZT)?

4. The query `select <datetime>'2023-05-18T10:56:00+09:00' - <datetime>'2020-09-10T05:00:00+00:00';` returns `{<duration>'23516:56:00'}` which is hard to read. How can we display how many days have passed without having to do the math ourselves?

5. How would you write the same query using `with` for each of the two times?

6. What's the best way to describe a type if you only want to see how you wrote it?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _One of the women vampires to her sisters: "He is young and strong; there are kisses for us all..."_
