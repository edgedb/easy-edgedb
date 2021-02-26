---
tags: Datetime, Describing Types
---

# Chapter 5 - Jonathan tries to leave the castle

Poor Jonathan is not having much luck. Here's what happens to him in this chapter:

> During the day, Jonathan decides to try to explore the castle but too many doors and windows are locked. He doesn't know how to get out, and wishes he could at least send Mina a letter. He pretends that there is no problem, and keeps talking to Dracula during the night. One night he sees Dracula climb out of his window and down the castle wall, like a snake. Now he is very afraid, and knows that Dracula is not human. A few days later he breaks one of the doors and finds another part of the castle. The room is very strange and he feels sleepy. When he opens his eyes, he sees three vampire women next to him. He is attracted to and afraid of them at the same time. He wants to kiss them, but knows that he will die if he does. They come closer, and he can't move...

## std::datetime

Since Jonathan was thinking of Mina back in London, let's learn about `std::datetime` because it uses time zones. To create a datetime, you can just cast a string in ISO 8601 format with `<datetime>`. That format looks like this:

`YYYY-MM-DDTHH:MM:SSZ`

And an actual date looks like this.

`'2020-12-06T22:12:10Z'`

The `T` inside there is just a separator, and the `Z` at the end stands for "zero timeline". That means that it is 0 different (offset) from UTC: in other words, it _is_ UTC.

One other way to get a `datetime` is to use the `to_datetime()` function. [Here is its signature](https://edgedb.com/docs/edgeql/funcops/datetime/#function::std::to_datetime), which shows that there are six ways to make a `datetime` with this function depending on how you want to make it. EdgeDB will know which one of the six you have chosen depending on what input you give it.

By the way, you'll notice one unfamiliar type inside called a [`decimal`](https://www.edgedb.com/docs/datamodel/scalars/numeric#type::std::decimal) type. This is a float with "arbitrary precision", meaning that you can give it as many numbers after the decimal point as you want. This is because float types on computers [become imprecise after a while](https://www.youtube.com/watch?v=-3c8G0JMM5Q) thanks to rounding errors. This example shows it:

```edgeql-repl
edgedb> SELECT 6.777777777777777; # Good so far
{6.777777777777777}
edgedb> SELECT 6.7777777777777777; # Add one more digit...
{6.777777777777778} # Where did the 8 come from?!
```

If you want to avoid this, add an `n` to the end to get a `decimal` type which will be as precise as it needs to be.

```edgeql-repl
edgedb> SELECT 6.7777777777777777n;
{6.7777777777777777n}
edgedb> SELECT 6.7777777777777777777777777777777777777777777777777n;
{6.7777777777777777777777777777777777777777777777777n}
```

Meanwhile, there is a `bigint` type that also uses `n` for an arbitrary size. That's because even int64 has a limit: it's 9223372036854775807.

```edgeql-repl
edgedb> SELECT 9223372036854775807; # Good so far...
{9223372036854775807}
edgedb> SELECT 9223372036854775808; # But add 1 and it will fail
ERROR: NumericOutOfRangeError: std::int64 out of range
```

So here you can just add an `n` and it will create a `bigint` that can accommodate any size.

```edgeql-repl
edgedb> SELECT 9223372036854775808n;
{9223372036854775808n}
```

Now that we know all the numeric types, let's get back to the six signatures for the `std::to_datetime` function:

```
std::to_datetime(s: str, fmt: OPTIONAL str = {}) -> datetime
std::to_datetime(local: cal::local_datetime, zone: str) -> datetime
std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime
std::to_datetime(epochseconds: decimal) -> datetime
std::to_datetime(epochseconds: float64) -> datetime
std::to_datetime(epochseconds: int64) -> datetime
```

The easiest is probably the third if you find ISO 8601 unfamiliar or you have a bunch of separate numbers to make into a date. With this, our game could have a function that generates integers for times that then use `to_datetime()` to get a proper time stamp.

Let's imagine that it's May 12. It's a bright morning at 10:35 in Castle Dracula. The sun is up, Dracula is asleep somewhere, and Jonathan is trying to use the time during the day to escape to send Mina a letter. In Romania the time zone is 'EEST' (Eastern European Summer Time). We'll use `to_datetime()` to generate this. We won't worry about the year, because the story takes place in the same year - we'll just use 2020 for convenience. We type this:

`SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');`

And get the following output:

`{<datetime>'2020-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

We can also use this to see the duration between events. EdgeDB has a `duration` type that you can get by subtracting a datetime from another one. Let's practice by calculating the exact number of seconds between one date in Central Europe and another in Korea:

```edgeql
SELECT to_datetime(2020, 5, 12, 6, 10, 0, 'CET') - to_datetime(2000, 5, 12, 6, 10, 0, 'KST');
```

This takes May 12 2020 6:10 am in Central European Time and subtracts May 12 2000 6:10 in Korean Standard Time. The result is: `{631180800s}`.

Now let's try something similar with Jonathan in Castle Dracula again, trying to escape. It's May 12 at 10:35 am. On the same day, Mina is in London at 6:10 am, drinking her morning tea. How many seconds passed between these two events? They are in different time zones but we don't need to calculate it ourselves; we can just specify the time zone and EdgeDB will do the rest:

```edgeql
SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST') - to_datetime(2020, 5, 12, 6, 10, 0, 'UTC');
```

The answer is 5100 seconds: `{5100s}`.

To make the query easier for us to read, we can also use the `WITH` keyword to create variables. We can then use the variables in `SELECT` below. We'll make one called `jonathan_wants_to_escape` and another called `mina_has_tea`, and subtract one from another to get a `duration`. With variable names it is now a lot clearer what we are trying to do:

```edgeql
WITH
  jonathan_wants_to_escape := to_datetime(2020, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := to_datetime(2020, 5, 12, 6, 10, 0, 'UTC'),
SELECT jonathan_wants_to_escape - mina_has_tea;
```

The output is the same: `{5100s}`. As long as we know the timezone, the `datetime` type does the work for us when we need a `duration`.

## Casting to a duration

Besides subtracting a `datetime` from another `datetime`, you can also just cast to make a `duration`. To do this, just write the number followed by the unit: `microseconds`, `milliseconds`, `seconds`, `minutes`, or `hours`. It will return a number of seconds, or a more precise unit if necessary. For example, `SELECT <duration>'2 hours`; will return `{7200s}`, and `SELECT <duration>'2 microseconds';` will return `{2µs}`.

You can include multiple units as well. For example:

```edgeql
SELECT <duration>'6 hours 6 minutes 10 milliseconds 678999 microseconds';
```

This will return `{21960.688999s}`.

EdgeDB is pretty forgiving when it comes to inputs when casting to a `duration`, and will ignore plurals and other signs. So even this horrible input will work:

```edgeql
SELECT <duration>'1 hours, 8 minute ** 5 second ()()()( //// 6 milliseconds' -
  <duration>'10 microsecond 7 minutes %%%%%%% 10 seconds 5 hour';
```

The result: `{-14344.99401s}`.

## Required links

Now we need to make a type for the three female vampires. We'll call it `MinorVampire`. These have a link to the `Vampire` type, which needs to be `required`. This is because Dracula controls them and they only exist as `MinorVampire`s because he exists.

```sdl
type MinorVampire extending Person {
  required link master -> Vampire;
}
```

Now that it's required, we can't insert a `MinorVampire` with just a name. It will give us this error: `ERROR: MissingRequiredError: missing value for required link default::MinorVampire.master`. So let's insert one and connect her to Dracula:

```edgeql
INSERT MinorVampire {
  name := 'Woman 1',
  master := (SELECT Vampire Filter .name = 'Count Dracula'),
};
```

This works because there is only one 'Count Dracula' (remember, `required link` is short for `required single link`). If there were more than one `Vampire`, we would have to add `LIMIT 1`. Without `LIMIT 1` we would get the following error:

```
error: possibly more than one element returned by an expression for a computable link 'master' declared as 'single'
```

## DESCRIBE to look inside types

Our `MinorVampire` type extends `Person`, and so does `Vampire`. Types can continue to extend other types, and they can extend more than one type at the same time. The more you do this, the more annoying it can be to try to combine it all together in your mind. This is where `DESCRIBE` can help, because it shows exactly what any type is made of. There are three ways to do it:

- `DESCRIBE TYPE MinorVampire` - this will give the [DDL (data definition language)](https://www.edgedb.com/docs/edgeql/ddl/index/) description of a type. DDL is a lower level language than SDL, the language we have been using. It is less convenient for schema, but is more explicit and can be useful for quick changes. We won't be learning any DDL in this course but later on you might find it useful sometimes. For example, with it you can quickly create functions without needing to do a migration. And if you understand SDL it will not be hard to pick up some tricks in DDL.

Now back to `DESCRIBE TYPE` which gives the results in DDL. Here's what our `Person` type looks like:

```
{
  'CREATE TYPE default::MinorVampire EXTENDING default::Person {
    CREATE REQUIRED SINGLE LINK master -> default::Vampire;
  };',
}
```

The `CREATE` keyword shows that it's a series of quick commands, which is why the order is important. In other words, SDL is _declarative_ (it _declares_ what something will be without worrying about order), while DDL is _imperative_ (it's a series of commands to change the state). Also, because it only shows the DDL commands to create it, it doesn't show us all the `Person` links and properties that it extends. So we don't want that. The next method is:

- `DESCRIBE TYPE MinorVampire AS SDL` - same thing, but in SDL.

The output is almost the same too, just the SDL version of the above. It's also not enough information for what we want now:

```
{
  'type default::MinorVampire extending default::Person {
    required single link master -> default::Vampire;
  };',
}
```

The third method is `DESCRIBE TYPE MinorVampire AS TEXT`. This is what we want, because it shows everything inside the type, including from the types that it extends. Here's the output:

```
{
  'type default::MinorVampire extending default::Vampire {
    required single link __type__ -> schema::Type {
        readonly := true;
    };
    optional single link lover -> default::Person;
    required single link master -> default::Vampire;
    optional multi link places_visited -> default::Place;
    optional single property age -> std::int16;
    required single property id -> std::uuid {
        readonly := true;
    };
    required single property name -> std::str;
  };',
}
```

(Note: `AS TEXT` doesn't include constraints and annotations. To see those, add `VERBOSE` at the end: `DESCRIBE TYPE MinorVampire AS TEXT VERBOSE;`. You'll learn about annotations in Chapter 14.)

The parts that say `readonly := true` we don't need to worry about, as they are automatically generated (and we can't touch them). For everything else, we can see that we need a `name` and a `master`, and could add a `lover`, `age` and `places_visited` for these `MinorVampire`s.

And for a _really_ long output, try typing `DESCRIBE SCHEMA` or `DESCRIBE MODULE default` (with `AS SDL` or `AS TEXT` if you want). You'll get an output showing the whole schema we've built so far.

So if `TYPE` comes after `DESCRIBE` for types and `MODULE` after `DESCRIBE` for modules, then what about links and all the rest? Here's the full list of keywords that can come after describe: `OBJECT`, `ANNOTATION`, `CONSTRAINT`, `FUNCTION`, `LINK`, `MODULE`, `PROPERTY`, `SCALAR TYPE`, `TYPE`. If you don't want to remember them all, just go with `OBJECT`: it will match anything inside your schema (except modules).

[Here is all our code so far up to Chapter 5.](code.md)

<!-- quiz-start -->

## Time to practice

1. What do you think `SELECT to_datetime(3600);` will return, and why?

   Hint: check the function signatures above and see which one EdgeDB will pick when you enter 3600.

2. Will `SELECT <int16>9 + 1.06n IS decimal;` work? And if it does, will it return `{true}`?

3. How many seconds went by between 5:00 am on Christmas Day 2003 in Turkmenistan (TMT) and 7:00 pm on New Year's Eve for the same year in Uzbekistan (UZT)?

4. How would you write the same query using `WITH` for each of the two times?

5. What's the best way to describe a type if you only want to see how you wrote it?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 6: [One of the women vampires to her sisters: "He is young and strong; there are kisses for us all."](../chapter6/index.md)
