---
tags: Ddl, Sdl, Edgedb Community
---

# Chapter 20 - The final battle

You made it to the final chapter - congratulations! Here's the final scene from the last chapter, though we won't spoil the final ending:

> Mina is almost a vampire now, and says she can feel Dracula all the time, no matter what hour of the day. Van Helsing arrives at Castle Dracula and Mina waits outside. Van Helsing then goes inside and destroys the vampire women and Dracula's coffin. Meanwhile, the other men approach from the south and are also close to Castle Dracula. Dracula's friends have him inside his box, and are carrying him on a wagon towards the castle as fast as they can. The sun is almost down, it is snowing, and the need to hurry to catch him. They get closer and closer, and grab the box. They pull the nails back and open it up, and see Dracula lying inside. Jonathan pulls out his knife. But just then the sun goes down. Dracula smiles and opens his eyes, and...

If you're curious about the ending to this scene, just [check out the book on Gutenberg](http://www.gutenberg.org/files/345/345-h/345-h.htm#CHAPTER_XIX) and search for "the look of hate in them turned to triumph".

We are sure that the vampire women have been destroyed, however, so we can do one final change by giving them a `last_appearance`. Van Helsing destroys them on November 5, so we will insert that date. But don't forget to filter Lucy out - she's the only `MinorVampire` that isn't one of the three women at the castle.

```edgeql
UPDATE MinorVampire FILTER .name != 'Lucy'
SET {
  last_appearance := <cal::local_date>'1887-11-05'
};
```

Depending on what happens in the last battle, we might have to do the same for Dracula or some of the heroes...

## Reviewing the schema

[Here's the schema and inserted data we have up to Chapter 20.](code.md)

Now that you've made it through 20 chapters, you should have a good understanding of the schema that we put together and how to work with it. Let's take a look at it one more time from top to bottom. We'll make sure that we fully understand it and think about which parts are good, and which need improvement, for an actual game.

The first part to a schema is always the command to start the migration:

- We've used `START MIGRATION TO {};` commands, but the {ref}` ``edgedb migration`` <docs:ref_cli_edgedb_migration>` tools provide better control in real projects.
- `module default {}`: We only used one module (namespace) for our schema, but you can make more if you like. You can see the module when you use `DESCRIBE TYPE AS SDL` (or `AS TEXT`).

Here's an example with `Person`, which starts like this and shows us the module it's located in:

`abstract type default::Person`

For a real game our schema would probably be a lot larger with various modules. We might see types in different modules like `abstract type characters::Person` and `abstract type places::Place`.

Our first type is called `HasNameAndCoffins`, which is abstract because we don't want any actual objects of this type. Instead, it is extended by types like `Place` because every place in our game

1. has a name, and
2. has a number of coffins (which is important because places without coffins are safer from vampires).

```sdl
abstract type HasNameAndCoffins {
  required property coffins -> int16 {
    default := 0;
  }
  required property name -> str {
    constraint exclusive;
    constraint max_len_value(30);
  }
}
```

We could have gone with {eql:type}` ``int32`` <docs:std::int32>`, {eql:type}` ``int64`` <docs:std::int64>` or {eql:type}` ``bigint`` <docs:std::bigint>` for the `coffins` property but we probably won't see that many coffins so {eql:type}` ``int16`` <docs:std::int16>` is fine.

Next is `abstract type Person`. This type is by far the largest, and does most of the work for all of our characters. Fortunately, all vampires used to be people and can have things like `name` and `age`, so they can extend from it too.

```sdl
abstract type Person {
  property first -> str;
  property last -> str;
  property title -> str;
  property degrees -> str;
  required property name -> str {
    constraint exclusive
  }
  property age -> int16;
  property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
  property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;
  property strength -> int16;
  multi link places_visited -> Place;
  multi link lover -> Person;
  property first_appearance -> cal::local_date;
  property last_appearance -> cal::local_date;
}
```

`exclusive` is probably the most common {ref}`constraint <docs:ref_datamodel_constraints>`, which we use to make sure that each character has a unique name. This works because we already know all the names of all the `NPC` types. But if there is a chance of more than one "Jonathan Harker" or other character, we could use the built-in `id` property instead. This built-in `id` is generated automatically and is already exclusive.

Properties like `conversational_name` are {ref}`computed properties <docs:ref_datamodel_computables>`. In our case, we added properties like `first` and `last` later on. It is tempting to remove `name` and only use `first` and `last` for every character, but the book has too many characters with strange names: `Woman 2`, `The innkeeper`, etc. In a standard user database, we would certainly only use `first` and `last` and a field like `email` with `constraint exclusive` to make sure that all users are unique.

Every property has a type (like `str`, `bigint`, etc.). Computed properties have them too but we don't need to tell EdgeDB the type because the computed expression itself tells the type. For example, `pen_name` takes `.name` which is a `str` and adds more strings, which will of course produce a `str`. The `++` used to join them together is called {eql:op}`concatenation <docs:STRPLUS>`.

The two links are `multi link`s, without which a `link` is to only one object. If you just write `link`, it will be a `single link`. It means that you may need to add `assert_single()` when creating a link or it will give this error:

```
error: possibly more than one element returned by an expression for a computed link 'former_self' declared as 'single'
```

This could be what you want for `lover`, but it wouldn't work well for `places_visited`.

For `first_appearance` and `last_appearance` we use {eql:type}`docs:cal::local_date` because our game is only based in one part of Europe inside a certain period. For a modern user database we would prefer {eql:type}`docs:std::datetime` because it is timezone aware and always ISO8601 compliant.

So for databases with users around the world, `datetime` is usually the best choice. Then you can use a function like {eql:func}`docs:std::to_datetime` to turn five `int64`s, one `float64` (for the seconds) and one `str` (for [the timezone](https://en.wikipedia.org/wiki/List_of_time_zone_abbreviations)) into a `datetime` that is always returned as UTC:

```edgeql-repl
edgedb> SELECT std::to_datetime(2020, 10, 12, 15, 35, 5.5, 'KST');
....... # October 12 2020, 3:35 pm and 5.5 seconds in Korea (KST = Korean Standard Time)
{<datetime>'2020-10-12T06:35:05.500Z'} # The return value is UTC, 6:35 (plus 5.5 seconds) in the morning
```

A similar abstract type to `HasNameAndCoffins` is this one:

```sdl
abstract type HasNumber {
  required property number -> int16;
}
```

We only used this for the `Crewman` type, which only extends two abstract types and nothing else:

```sdl
type Crewman extending HasNumber, Person {
}
```

This `HasNumber` type was used for the five `Crewman` objects, who in the beginning didn't have a name. But later on, we used those numbers to create names based on the numbers:

```edgeql
UPDATE Crewman
SET {
  name := 'Crewman ' ++ <str>.number
};
```

So even though it was rarely used, it could become useful later on. For types later in the game you could imagine this being used for townspeople or random NPCs: 'Shopkeeper 2', 'Carriage Driver 12', etc.

Our vampire types extend `Person`, while `MinorVampire` also has an optional (single) link to `Person`. This is because some characters begin as humans and are "reborn" as vampires. With this format, we can use the properties `first_appearance` and `last_appearance` from `Person` to have them appear in the game. And if one is turned into a `MinorVampire`, we can link the two.

```sdl
type Vampire extending Person {
  multi link slaves -> MinorVampire;
}

type MinorVampire extending Person {
  link former_self -> Person;
}
```

With this format we can do a query like this one that pulls up all people who have turned into `MinorVampire`s.

```edgeql
SELECT Person {
  name,
  vampire_name := .<former_self[IS MinorVampire].name
} FILTER EXISTS .vampire_name;
```

In our case, that's just Lucy: `{default::NPC {name: 'Lucy Westenra', vampire_name: {'Lucy'}}}` But if we wanted to, we could extend the game back to more historical times and link the vampire women to an `NPC` type. That would become their `former_self`.

Our two enums were used for the `PC` and `Sailor` types:

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
type Sailor extending Person {
  property rank -> Rank;
}

scalar type Transport extending enum<Feet, HorseDrawnCarriage, Train>;
type PC extending Person {
  required property transport -> Transport;
}
```

The enum `Transport` never really got used, and needs some more transportation types. We didn't look at these in detail, but in the book there are a lot of different types of transport. In the last chapter, Arthur's team that waited at Varna used a boat called a "steam launch" which is smaller than the boat "The Demeter", for example. This enum would probably be used in the game logic itself in this sort of way:

- choosing `Feet` gives the character a certain speed and costs nothing,
- `HorseDrawnCarriage` increases speed but decreases money,
- `Train` increases speed the most but decreases money and can only follow railway lines, etc.

`Visit` is one of our two "hackiest" (but most fun) types. We stole most of it from the `Time` type that we created earlier but never used. In it, we have a `time` property that is just a string, but gets used in this way:

- by casting it into a {eql:type}`docs:cal::local_time` to make the `local_time` property,
- by slicing its first two characters to get the `hour` property, which is just a string. This is only possible because we know that even single digit numbers like `1` need to be written with two digits: `01`
- by another computed property called `awake` that is either 'asleep' or 'awake' depending on the `hour` property we just made, cast into an `int16`.

```sdl
type Visit {
  required link ship -> Ship;
  required link place -> Place;
  required property date -> cal::local_date;
  property time -> str;
  property local_time := <cal::local_time>.time;
  property hour := .time[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

The NPC type is where we first saw the {ref}` ``overloaded`` <docs:ref_eql_sdl_links_overloading>` keyword, which lets us use properties, links, functions etc. in different ways than the default. Here we wanted to constrain `age` to 120 years, and to use the `places_visited` link in a different way than in `Person` by giving it London as the default.

```sdl
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

Our `Place` type shows that you can extend as many times as you want. It's an `abstract type` that extends another `abstract type`, and then gets extended for other types like `City`.

```sdl
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

The `important_places` property only got used once in this insert:

```edgeql
INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

and right now it is just an array. We can keep it unchanged for now, because we haven't made a type yet for really small locations like hotels and parks. But if we do make a new type for these places, then we should turn it into a `multi link`. Even our `OtherPlace` type is not quite the right type for this, as the {ref}`annotation <docs:ref_eql_sdl_annotations>` shows:

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns count! Use the Castle type for that';
}
```

So in a real game we would create some other smaller location types and make them a link from the `property important_places` inside `City`. We might also move `important_places` to `Place` so that types like `Region` could link from it too.

Annotations: we used `abstract annotation` to add a new annotation:

```sdl
abstract annotation warning;
```

because by default a type {ref}`can only have annotations <docs:ref_datamodel_annotations>` called `title`, `description`, or `deprecated`. We only used annotations for fun for this one type, because nobody else is working on our database yet. But if we made a real database for a game with many people working on it, we would put annotations everywhere to make sure that they know how to use each type.

Our `Lord` type was only created to show how to use `constraint expression on`, which lets us make our own constraints:

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord') = true) {
    errmessage := "All lords need \'Lord\' in their name";
  };
};
```

(We might remove this in a real game, or maybe it would become type Lord extending PC so player characters could choose to be a lord, thief, detective, etc. etc.)

The `Lord` type uses the function {eql:func}`docs:std::contains` which returns `true` if the item we are searching for is inside the string, array, etc. It also uses `__subject__` which refers to the type itself: `__subject__.name` means `Person.name` in this case. {eql:constraint}`Here are some more examples <docs:std::expression>` from the documentation of using `constraint expression on`.

Another possible way to create a `Lord` is to do it this way, since `Person` has the property called `title`:

```sdl
type Lord extending Person {
  constraint expression on (__subject__.title = 'Lord') {
    errmessage := "All lords need \'Lord\' in their name";
  };
}
```

This will depend on if we want to create `Lord` types with names just as a single string in `.name`, or by using `.first`, `.last`, `.title` etc. with a computed property to form the full name.

Our next types extending `Place` including `Country` and `Region` were looked at just last chapter, so we won't review them here. But `Castle` is a bit unique:

```sdl
type Castle extending Place {
  property doors -> array<int16>;
}
```

Back in Chapter 7, we used this in a query to see if Jonathan could break any of the doors and escape the castle. The idea was simple: Jonathan would try to open every door, and if he had more strength then any one of them then he could escape the castle.

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT jonathan_strength > min(array_unpack(doors));
```

However, later on we learned the `any()` function so let's see how we could use it here. With `any()`, we could change the query to this:

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT any(array_unpack(doors) < jonathan_strength); # Only this part is different
```

And of course, we could also create a function to do the same now that we know how to write functions and how to use `any()`. Since we are filtering by name (Jonathan Harker and Castle Dracula), the function would also just take two strings and do the same query.

Don't forget, we needed {eql:func}`docs:std::array_unpack` because the function {eql:func}`docs:std::any` works on sets:

```sdl
std::any(values: SET OF bool) -> bool
```

So this (a set) will work: `SELECT any({5, 6, 7} = 7);`

But this (an array) will not: `SELECT any([5, 6, 7] = 7);`

Our next type is `BookExcerpt`, which we imagined being useful for the humans creating the database. It would need a lot of inserts from each part of the book, with the text exactly as written. Because of that, we chose to use {ref}` ``index on`` <docs:ref_eql_sdl_indexes>` for the `excerpt` property, which will then be faster to look up. Remember to use this only where needed: it will increase lookup speed, but make the database larger overall.

```sdl
type BookExcerpt {
  required property date -> cal::local_datetime;
  required link author -> Person;
  required property excerpt -> str;
  index on (.excerpt);
}
```

Next is our other fun and hacky type, `Event`.

```sdl
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  multi link excerpt -> BookExcerpt;
  property exact_location -> tuple<float64, float64>;
  property east -> bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ ('E' IF .east = true ELSE 'W');
}
```

This one is probably closest to an actual usable type for a real game. With `start_time` and `end_time`, `place` and `people` (plus `url`) we can properly arrange which characters are at which locations, and when. The `description` property is for users of the database with descriptions like `'The Demeter arrives at Whitby, crashing on the beach'` that are used to find events when we need to.

The last two types in our schema, `Currency` and `Pound`, were created two chapters ago so we won't review them here.

## Navigating EdgeDB documentation

Now that you have reached the end of the book, you will certainly start looking at the EdgeDB documentation. We'll close the book out with some tips to do so, so that it feels familiar and easy to look through.

### Syntax

This book included a lot of links to EdgeDB documentation, such as types, functions, and so on. If you are trying to create a type, property etc. and are having trouble, a good idea is to start with the section on syntax. This section always shows the order you need to follow, and all the options you have.

For a simple example, {ref}`here is the syntax on creating a module <docs:ref_eql_sdl_modules>`:

```sdl-synopsis
module ModuleName "{"
  [ schema-declarations ]
  ...
"}"
```

Looking at that you can see that a module is just a module name, `{}`, and everything inside (the schema declarations). Easy enough.

How about object types? {ref}`They look like this <docs:ref_eql_sdl_object_types>`:

```sdl-synopsis
[abstract] type TypeName [extending supertype [, ...] ]
[ "{"
    [ annotation-declarations ]
    [ property-declarations ]
    [ link-declarations ]
    [ constraint-declarations ]
    [ index-declarations ]
    ...
  "}" ]
```

This should be familiar to you: you need `type TypeName` to start. You can add `abstract` on the left and `extending` for other types, and then everything else goes inside `{}`.

Meanwhile, the {ref}`properties are more complex <docs:ref_eql_sdl_props>` and include three types: concrete, computed, and abstract. We're most familiar with concrete so let's take a look at that:

```sdl-synopsis
[ overloaded ] [{required | optional}] [{single | multi}]
  property name
  [ extending base [, ...] ] -> type
  [ "{"
      [ default := expression ; ]
      [ readonly := {true | false} ; ]
      [ annotation-declarations ]
      [ constraint-declarations ]
      ...
    "}" ]
```

You can think of the syntax as a helpful guide to keep your declarations in the right order.

### Dipping into DDL

DDL is something you'll see mainly when dealing with migrations because it's good for expressing incremental changes. Up to now, we've only mentioned DDL for functions because it's so easy to just add `CREATE` to make a function whenever you need.

SDL: `function says_hi() -> str using('hi');`

DDL: `CREATE FUNCTION says_hi() -> str USING('hi')`

And even the capitalization doesn't matter.

But for types, DDL requires a lot more typing, using keywords like `CREATE`, `SET`, `ALTER`, and so on. Using {ref}` ``edgedb migration`` <docs:ref_cli_edgedb_migration>` tools makes it possible to work with the schema using only SDL.

## EdgeDB lexical structure

You might want to take a look at or bookmark {ref}`this page <docs:ref_eql_lexical>` for reference during your projects. It contains the whole lexical structure of EdgeDB including items that are maybe too dry for a textbook like this one. Things like order of precedence for operators, all reserved keywords, which characters can be used in identifiers, and so on.

## Getting help

Help is always just a message away. The best way to get help is to start a discussion on [our discussion board](https://github.com/edgedb/edgedb/discussions) on GitHub. You can also [start an issue here](https://github.com/edgedb/edgedb/issues/new/choose) on EdgeDB, or do the same for the Easy EdgeDB book on this very page.

## And now it's time to say goodbye

We hope you enjoyed learning EdgeDB through a story, and are now familiar enough with it to implement it for your own projects. Ironically, if we wrote the book with enough detail to answer all your questions then we might never see you on the forums! If that's the case, then we wish you the best of luck with your projects. Let's finish the book up with a poem from another book, the Lord of the Rings, on the endless possibilities of life.

> The Road goes ever on and on
> Down from the door where it began.
> Now far ahead the Road has gone,
> And I must follow, if I can,
> Pursuing it with eager feet,
> Until it joins some larger way
> Where many paths and errands meet.
> And whither then? I cannot say.

See you, or not see you, however things turn out! Thanks again for reading.
