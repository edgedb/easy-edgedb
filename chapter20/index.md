---
tags: Ddl, Sdl, Edgedb Community
---

# Chapter 20 - The final battle

You made it to the final chapter, congratulations! Here's the final scene from the last chapter, though we won't spoil the final ending in case you now plan to read the original book:

> Mina is almost a vampire now, and says she can feel Dracula all the time, no matter what hour of the day. Van Helsing arrives at Castle Dracula and Mina waits outside. Van Helsing then goes inside and destroys the vampire women and Dracula's coffin.
>
> At the same time the four other heroes (Jonathan Harker, Quincy Morris, Doctor Seward, Arthur Holmwood) are approaching from the south to try to catch up to Dracula. The sun is still up, so Dracula is inside his box protected from the sunlight on a wagon that his friends are taking to the castle as fast as they can. The sun is almost down, it is snowing, and our heroes need to hurry to catch him. A fight ensues, and the heroes drive back Dracula's friends and finally reach the box. They pull the nails back and open it up, and see Dracula lying inside. Jonathan pulls out his knife. But just then the sun goes down and night begins. Dracula smiles and opens his eyes, and...

If you're curious about the ending to this scene, just [check out the book on Gutenberg](https://www.gutenberg.org/files/345/345-h/345-h.htm#chap19) and search for "the look of hate in them turned to triumph".

We are sure that the vampire women have been destroyed, however, so we can do one final change by giving them a `last_appearance`. Van Helsing destroys them on November 5, so we will insert that date. But don't forget to filter Lucy out - she's the only `MinorVampire` that isn't one of the three women at the castle.

```edgeql
update MinorVampire filter .name != 'Lucy'
set {
  last_appearance := <cal::local_date>'1893-11-05'
};
```

Depending on what happens in the last battle, we might have to do the same for Dracula or some of the heroes...

## Reviewing the schema

[Here's the schema and inserted data we have up to Chapter 20.](code.md)

Now that you've made it through all 20 chapters, you should have a good understanding of the schema that we put together and how to work with it. Let's take a look at it one more time from top to bottom. We'll make sure that we fully understand it and think about which parts are good, and which need improvement, for an actual game.

First let's start with the schema in general.

- To migrate a schema, just use the {ref}` ``edgedb migration`` <docs:ref_cli_edgedb_migration>` tools. A simple `edgedb migration create` and `edgedb migrate` is all you need to migrate your schema to a new one.
- `module default {}`: We only used one module (namespace) for our schema, but you can make more modules if you like. You can see the module when you use `describe type as sdl` (or `as text`).

Here's an example with `Person`, which starts like this and shows us the module it's located in:

`abstract type default::Person`

For a real game our schema would probably be a lot larger with various modules. We might see types in different modules like `abstract type characters::Person` and `abstract type places::Place`.

Our first type is called `HasNameAndCoffins`, which is abstract because we don't want any actual objects of this type. Instead, it is extended by types like `Place` because every place in our game:

1. has a name, and
2. has a number of coffins (which is important because places without coffins are safer from vampires).

```sdl
abstract type HasNameAndCoffins {
  required coffins: int16 {
    default := 0;
  }
  required name: str {
    delegated constraint exclusive;
    constraint max_len_value(30);
  }
}
```

We could have gone with {eql:type}` ``int32`` <docs:std::int32>`, {eql:type}` ``int64`` <docs:std::int64>` or {eql:type}` ``bigint`` <docs:std::bigint>` for the `coffins` property but we probably won't see that many coffins so {eql:type}` ``int16`` <docs:std::int16>` is fine.

Next is `abstract type Person`. This type is by far the largest, and does most of the work for all of our characters. Fortunately, all vampires used to be people and can have things like `name` and `age`, so they can extend from it too.

```sdl
abstract type Person extending HasMoney {
  required name: str {
    delegated constraint exclusive;
  }
  multi places_visited: Place;
  multi lovers: Person;
  is_single := not exists .lovers;
  strength: int16;
  first_appearance: cal::local_date;
  last_appearance: cal::local_date;
  age: int16;
  title: str;
  degrees: array<str>;
  conversational_name := .title ++ ' ' 
    ++ .name if exists .title else .name;
  pen_name := .name ++ ', ' 
    ++ array_join(.degrees, ', ') if exists .degrees else .name;
}
```

`exclusive` is probably the most common {ref}`constraint <docs:ref_datamodel_constraints>`, which we use to make sure that each character has a unique name. This works because we already know all the names of all the `NPC` types. But if there is a chance of more than one "Jonathan Harker" or other character, we could use the built-in `id` property instead. This built-in `id` is generated automatically and is already exclusive.

Properties like `conversational_name` are {ref}`computed properties <docs:ref_datamodel_computed>`. In our case, we added properties like `first` and `last` later on. It is tempting to remove `name` and only use `first` and `last` for every character, but the book has too many characters with names that wouldn't fit this like `Woman 2` and `The innkeeper`. In a standard database used to record the data for users of an app, we would certainly only use `first` and `last` and a field like `email` with `constraint exclusive` to make sure that all users are unique.

Every property has a type (like `str`, `bigint`, etc.). Computed properties have them too but we don't need to tell EdgeDB the type because the computed expression itself tells the type. For example, `pen_name` takes `.name` which is a `str` and adds more strings, which will of course produce a `str`. The `++` used to join them together is called {eql:op}`concatenation <docs:strplus>`. On the other hand, with a computed property you do have to tell EdgeDB whether it is a `property` or a `link`.

The two links on the `Person` type are `multi` links. Without the keyword `multi` it will be a single link and can only link to a single other object. This means that you may need to add `assert_single()` when creating a link or it will give this error:

```
error: possibly more than one element returned by an expression
for a computed link 'former_self' declared as 'single'
```

On the other hand, backlinks have the opposite behavior: a backlink is a `multi` link by default, meaning that you have to write `single` otherwise.

For `first_appearance` and `last_appearance` we use {eql:type}`docs:cal::local_date` because our game is only based in one part of Europe inside a certain period. For a modern user database we would prefer {eql:type}`docs:std::datetime` because it is timezone aware and ISO8601 compliant.

So for databases with users around the world, `datetime` is usually the best choice. Then you can use a function like {eql:func}`docs:std::to_datetime` to turn five `int64`s, one `float64` (for the seconds) and one `str` (for [the timezone](https://en.wikipedia.org/wiki/List_of_time_zone_abbreviations)) into a `datetime` that is always returned as UTC:

```
db> select std::to_datetime(2020, 10, 12, 15, 35, 5.5, 'KST');
....... # October 12 2020, 3:35 pm and 5.5 seconds in Korea (KST = Korean Standard Time)
{<datetime>'2020-10-12T06:35:05.500Z'} # The return value is UTC, 6:35 (plus 5.5 seconds) in the morning
```

A similar abstract type to `HasNameAndCoffins` is this one:

```sdl
abstract type HasNumber {
  required number: int16;
}
```

We only used this for the `Crewman` type, which only extends two abstract types and nothing else:

```sdl
type Crewman extending HasNumber, Person {
  overloaded name: str {
    default := 'Crewman ' ++ <str>.number;
  }
}
```

This `HasNumber` type was used for the five `Crewman` objects, who don't have names. Because their `name` property inherited from `Person` is required, we used their `number` property to give them each a name: Crewman 1, Crewman 2, and so on.

```edgeql
for n in {1, 2, 3, 4, 5}
  union (
    insert Crewman {
      number := n,
      first_appearance := cal::to_local_date(1893, 7, 6),
      last_appearance := cal::to_local_date(1893, 7, 16),
    }
  );
```

So even though `HasNumber` was rarely used, it could become useful later on. For types later in the game you could imagine this being used for townspeople or random NPCs: 'Shopkeeper 2', 'Carriage Driver 12', etc.

Our vampire types extend `Person`, while `MinorVampire` also has a single and non-required link to `Person`. This is because some characters begin as humans and are "reborn" as vampires. With this format, we can use the properties `first_appearance` and `last_appearance` from `Person` to have them appear in the game. And if one is turned into a `MinorVampire`, we can link the two.

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on source delete delete target;
    property combined_strength := (Vampire.strength + .strength) / 2;
  }
  army_strength := sum(.slaves@combined_strength);
}

type MinorVampire extending Person {
  former_self: Person;
  single master := assert_single(.<slaves[is Vampire]);
  master_name := .master.name;
};
```

With this format we can do a query like this one that pulls up all people who have turned into `MinorVampire`s.

```edgeql
select Person {
  name,
  vampire_name := .<former_self[is MinorVampire].name
} filter exists .vampire_name;
```

In our case, that's just Lucy: `{default::NPC {name: 'Lucy Westenra', vampire_name: {'Lucy'}}}` But if we wanted, we could extend the game back before the events of the book and link the vampire women to an `NPC` type. That would become their `former_self`.

The `PC` and `Sailor` types show two enums and one sequence that we used:

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

type Sailor extending Person {
  rank: Rank;
}

scalar type Class extending enum<Rogue, Mystic, Merchant>;

scalar type PCNumber extending sequence;

type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement();
  }
  required number: PCNumber {
    default := sequence_next(introspect PCNumber);
  }
  multi party: Party {
    on source delete delete target if orphan;
    on target delete allow;
  }
  overloaded required name: str {
    constraint max_len_value(30);
  }
  last_updated: datetime {
    rewrite insert, update using (datetime_of_statement());
  }
  bonus_item: LotteryTicket {
    rewrite insert, update using (get_ticket());
  }
}
```

The `PCNumber` type has been quite useful, allowing us to keep track of how many `PC` objects have been created even if some of them get deleted later. If you end up adding and deleting a lot of `PC` objects then the following query will show pretty different numbers between the latest sequence number and the total number of `PC` objects:

```edgeql
with latest := (select <str>max(PC.number)),
select {'Total PCs created: ' ++ latest ++ ' Current PCs: ' ++ <str>count(PC) };
```

`ShipVisit` is one of our two "hackiest" (but most fun) types. We stole some of it from the `Time` type that we created earlier and later decided to turn into a single global object. Inside the `ShipVisit` type we have a `clock` property that is just a string, but gets used in this way:

- by casting it into a {eql:type}`docs:cal::local_time` to make the `clock_time` property,
- by slicing its first two characters to get the `hour` property, which is just a string. This is only possible because we know that even single digit numbers like `1` need to be written with two digits: `01`
- by another computed property called `vampires_are` that is either `Asleep` or `Awake` depending on the `hour` property we just made, cast into an `int16`.

```sdl
type ShipVisit {
  required ship: Ship;
  required place: Place;
  required date: cal::local_date;
  clock: str;
  clock_time := <cal::local_time>.clock;
  hour := .clock[0:2];
  vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
        else SleepState.Awake;
}
```

The NPC type is where we first saw the {ref}` ``overloaded`` <docs:ref_eql_sdl_links_overloading>` keyword, which lets us use properties, links, functions etc. in different ways than in the type's parent type. Here we wanted to constrain `age` to 120 years, and to use the `places_visited` link in a different way than in `Person` by giving it London as the default.

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120)
  }
}
```

Our `Place` type shows that you can extend as many times as you want. It's an `abstract type` that extends another `abstract type`, and then gets extended for other types like `City`.

```sdl
abstract type Place extending HasNameAndCoffins {
  modern_name: str;
  multi important_places: Landmark;
}
```

The `important_places` property used to be an `<array<str>>`, but in Chapter 16 we decided to create a `Landmark` type that would represent the smallest type of location that would appear in the game such as hotels, parks, universities, and so on. These locations are so small that they don't need to track the number of coffins, because the number of coffins is relevant for larger spaces of land to help determine if the location can be easily terrorized by vampires or not.

Annotations: we used `abstract annotation` to add a new annotation:

```sdl
abstract annotation warning;
```

This was necessary because by default a type {ref}`can only have annotations <docs:ref_datamodel_annotations>` called `title`, `description`, or `deprecated`. We only used annotations for fun for this one type, because nobody else is working on our database yet. But if we made a real database for a game with many people working on it, we would put annotations everywhere to make sure that they know how to use each type.

Our `Lord` type was only created to show how to use `constraint expression on`, which lets us make our own constraints:

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord')) {
    errmessage := "All lords need \'Lord\' in their name";
  }
}
```

We might remove this in a real game, or maybe it would become `type Lord extending PC` so player characters could choose to be a lord, thief, detective, etc.

The `Lord` type uses the function {eql:func}`docs:std::contains` which returns `true` if the item we are searching for is inside the string, array, etc. It also uses `__subject__` which refers to the type itself: `__subject__.name` means `Person.name` in this case. {eql:constraint}`Here are some more examples <docs:std::expression>` from the documentation of using `constraint expression on`.

Another possible way to create a `Lord` is to do it this way, since `Person` has a property called `title`:

```sdl
type Lord extending Person {
  constraint expression on (__subject__.title = 'Lord') {
    errmessage := "All lords need \'Lord\' in their title";
  }
}
```

This will depend on if we want to create `Lord` types with names just as a single string in `.name`, or by using `.first`, `.last`, `.title` etc. with a computed property to form the full name.

Our next types extending `Place` including `Country` and `Region` were looked at just last chapter, so we won't review them here. But `Castle` is a bit unique:

```sdl
type Castle extending Place {
  doors: array<int16>;
}
```

Back in Chapter 7, we used this in a query to see if Jonathan could break any of the doors and escape the castle. The idea was simple: Jonathan would try to open every door, and if he had more strength then any one of them then he could escape the castle.

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  doors := (select Castle filter .name = 'Castle Dracula').doors,
select jonathan_strength > min(array_unpack(doors));
```

However, later on we learned the `any()` function so let's see how we could use it here. With `any()`, we could change the query to this:

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  doors := (select Castle filter .name = 'Castle Dracula').doors,
select any(array_unpack(doors) < jonathan_strength); # Only this part is different
```

And of course, we could also create a function to do the same now that we know how to write functions and how to use `any()`. Since we are filtering by name ('Jonathan Harker' and 'Castle Dracula'), the function would also just take two strings and do the same query.

Also don't forget that we needed {eql:func}`docs:std::array_unpack` because the function {eql:func}`docs:std::any` works on sets:

```sdl
std::any(values: set of bool) -> bool
```

So this (a set) will work: `select any({5, 6, 7} = 7);`

But this (an array) will not: `select any([5, 6, 7] = 7);`

Our next type is `BookExcerpt`, which we imagined being useful for the developers creating the database. It would need a lot of inserts from each part of the book, with the text exactly as written. We chose to use {ref}` ``index on`` <docs:ref_eql_sdl_indexes>` for the `date` property, which will then be faster when we need to order by date. Remember to use indexes only where needed: they speed up queries that filter, order, and group, but make the database larger overall and slow down inserts and updates.

```sdl
type BookExcerpt {
  required date: cal::local_datetime;
  required excerpt: str;
  index on (.date);
  required author: Person;
}
```

Next is our other fun and hacky type, `Event`.

```sdl
type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  location: tuple<float64, float64>;
  index on (.location);
  ns_suffix := '_N_' if .location.0 > 0.0 else '_S_';
  ew_suffix := '_E' if .location.1 > 0.0 else '_W';
  url := get_url() 
    ++ <str>(math::abs(.location.0)) ++ .ns_suffix 
    ++ <str>(math::abs(.location.1)) ++ .ew_suffix;
}
```

This one is probably closest to an actual usable type for a real game. With `start_time` and `end_time`, `place` and `people` (plus `url`) we can properly arrange which characters are at which locations, and when. The `description` property makes it easy for users of the database to find events. It might contain something like `'The Demeter arrives at Whitby, crashing on the beach'`.

And the output for the `Event` type is especially nice as JSON. You can imagine how useful this might be for our game setting:

```
{
  "id": "d80dde9c-fec9-11ed-9c27-bffd94675ea1",
  "description": "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  "east": false,
  "location": [54.4858, 0.6206],
  "url": "https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W54.4858_N_0.6206_W",
  "end_time": "1893-09-11T23:00:00",
  "start_time": "1893-09-11T18:00:00",
  "people": [
    {
      "strength": 4,
      "id": "f22b1910-fd08-11ed-ab09-9b95a39d5d69",
      "first_appearance": null,
      "last_appearance": "1893-09-20",
      "name": "Lucy Westenra",
      "age": null,
      "title": null,
      "conversational_name": "Lucy Westenra",
      "degrees": null,
      "pen_name": "Lucy Westenra"
    },
    {
      "strength": 2,
      "id": "dea9080a-fe8b-11ed-9759-f3313536553b",
      "first_appearance": null,
      "last_appearance": null,
      "name": "John Seward",
      "age": null,
      "title": null,
      "conversational_name": "John Seward",
      "degrees": null,
      "pen_name": "John Seward"
    },
    {
      "strength": 4,
      "id": "70bad6e0-fea2-11ed-97e2-ef3281250140",
      "first_appearance": null,
      "last_appearance": null,
      "name": "Abraham Van Helsing",
      "age": null,
      "title": "Dr.",
      "conversational_name": "Dr. Abraham Van Helsing",
      "degrees": "M.D., Ph. D. Lit., etc.",
      "pen_name": "Abraham Van Helsing, M.D., Ph. D. Lit., etc."
    }
  ],
  "place": [
    {
      "id": "d64af6d0-fec9-11ed-9c27-a3cf8be8d973",
      "important_places": null,
      "modern_name": null,
      "name": "Whitby",
      "coffins": 0
    }
  ],
  "excerpt": []
}
```

The last two types in our schema, `Currency` and `Pound`, were created just two chapters ago so they are still fresh in our mind. We won't need to review them here.

## Navigating EdgeDB documentation

Now that you have reached the end of the book, you'll be referencing our documentation to refresh what you know and to learn more. We'll close the book out with some tips to do so, so that it feels familiar and easy to look through.

### Syntax

This book included a lot of links to EdgeDB documentation, such as types, functions, and so on. If you are trying to create one of these items and are having trouble, a good idea is to start with the section on syntax. This section always shows the order you need to follow, and all the options you have. We looked at the syntax of a few items during this book, but there is a lot more of this to reference in the documentation.

For a simple example, {ref}`here is the syntax on creating a module <docs:ref_eql_sdl_modules>`:

```sdl-synopsis
module ModuleName "{"
  [ schema-declarations ]
  ...
"}"
```

Looking at that you can see that a module is just a module name, `{}`, and everything inside (the schema declarations). The `[]` square brackets in documentation are used to show optional input. In other words, a module can contain schema declarations or not. Easy enough.

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

This should be familiar to you: you need `type TypeName` to start. You can optionally add `abstract` on the left and `extending` for other types, and then everything else goes inside `{}`.

Meanwhile, the {ref}`properties are more complex <docs:ref_eql_sdl_props>` and include three types: concrete, computed, and abstract. We're most familiar with concrete so let's take a look at that:

```sdl-synopsis
[ overloaded ] [{required | optional}] [{single | multi}]
  [ property ] name : type
  [ "{"
      [ extending base [, ...] ; ]
      [ default := expression ; ]
      [ readonly := {true | false} ; ]
      [ annotation-declarations ]
      [ constraint-declarations ]
      ...
    "}" ]
```

The `{ | }` in documentation is used to show the possible options available to you. So `[{required | optional}]` means that you don't need to write either required or optional (because both are inside `[]` square brackets), but if you choose to use it, you must choose either `required` or `optional`, not both.

You can think of the syntax as a helpful guide to keep your declarations in the right order, and to give you ideas of the full range of possibilities.

### Dipping into DDL

We have only seen DDL in our `.edgeql` files that are automatically generated every time a migration takes place. DDL used to be used sometimes in the past, and was even mentioned in the first edition of this book. But with better and better migration tools, there is little need for it. And in fact, EdgeDB is set by default to disallow DDL once you have completed a migration with the project tools. Take this attempt to use DDL for example and the error output it generates:

```
db> create function hi() -> str using ("Hi");
error: QueryError: bare DDL statements are not allowed in this database
  ┌─ <query>:1:1
  │
1 │ create function hi() -> str using ("Hi");
  │ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Use the migration commands instead.
  │
  = The `allow_bare_ddl` configuration variable is set to 'NeverAllow'.
  The `edgedb migrate` command normally sets this to avoid accidental schema changes
  outside of the migration flow.
```

If you absolutely do want to use DDL, the configuration [can be temporarily changed](https://www.edgedb.com/docs/reference/configuration#query-behavior) until a migration is run. Otherwise, the most recommended way to interact with DDL is by gaining a passive knowledge of it in case you want to double check a migration script before applying it.

## EdgeDB lexical structure

You might want to take a look at or bookmark {ref}`this page <docs:ref_eql_lexical>` for reference during your projects. It contains the whole lexical structure of EdgeDB including items that are maybe too dry for a textbook like this one. This includes things like order of precedence for operators, all reserved keywords, which characters can be used in identifiers, and so on.

## Getting help

Help is always just a message away. The most active place to discuss EdgeDB and get help is [our Discord server](https://discord.gg/edgedb), while the [discussion board](https://github.com/edgedb/edgedb/discussions) on GitHub is another good place to start a conversation. You can also [start an issue here](https://github.com/edgedb/edgedb/issues/new/choose) on EdgeDB, or do the same for the Easy EdgeDB book on [its dedicated GitHub repo](https://github.com/edgedb/easy-edgedb/).

## EdgeDB Cloud

Your knowledge of EdgeDB and EdgeQL should be pretty good at this point, and you may be looking to take your EdgeDB-powered product to the next step. With [EdgeDB Cloud](https://www.edgedb.com/cloud) you can deploy your database instantly and connect from anywhere with near-zero configuration. Leave the infrastructure to us!

## And now it's time to say goodbye

We hope you enjoyed learning EdgeDB through a story and are now familiar enough with it to implement it for your own projects. Ironically, if we wrote the book with enough detail to answer all your questions then we might never see you on our Discord! If that's the case, then we wish you the best of luck with your projects. Let's finish the book up with a poem from another book, the Lord of the Rings, on the endless possibilities of life.

> The Road goes ever on and on
> Down from the door where it began.
> Now far ahead the Road has gone,
> And I must follow, if I can,
> Pursuing it with eager feet,
> Until it joins some larger way
> Where many paths and errands meet.
> And whither then? I cannot say.

See you, or not see you, however things turn out! Thanks again for reading.
