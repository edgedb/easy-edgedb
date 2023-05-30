---
tags: Defaults, Overloading, For Loops
---

# Chapter 9 - Strange events in England

The episode for this chapter is a flashback to see why everybody is in the town of Whitby in the first place. We've gone back in time a few weeks to when the ship left Varna and Mina and Lucy haven't left for Whitby yet. The introduction is also split into two parts. Here's the first:

> We still don't know where Jonathan is, and the ship The Demeter is on its way to England with Dracula inside. Meanwhile, Mina Harker is in London writing letters to her friend Lucy Westenra. Lucy has three boyfriends (named Dr. John Seward, Quincey Morris, and Arthur Holmwood), and they all want to marry her....

## Working with dates some more

It looks like we have some more people to insert. But first, let's think about the ship a little more. Everyone on the ship was killed by Dracula, but we don't want to delete the crew because they are still part of our game. The book tells us that the ship left on the 6th of July, and the last person (the captain) died on the 4th of August (in 1893).

This is a good time to add two new properties to the `Person` type to indicate when a character is present. We'll call them `first_appearance` and `last_appearance`. The name `last_appearance` is a bit better than `death`, because for the game it doesn't matter: we just want to know when characters are there or not.

For these two properties we will just use `cal::local_date` for the sake of simplicity. There is also `cal::local_datetime` that includes time, but we should be fine with just the date. (And of course there is the `cal::local_time` type with just the time of day that we have in our `Date` type.)

Before we used the function `std::to_datetime` which took seven parameters; this time we'll use a similar but shorter {eql:func}`docs:cal::to_local_date` function. It just takes three integers.

Here are its signatures (we're using the third):

```
cal::to_local_date(s: str, fmt: optional str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

So let's do a migration now that these `first_appearance` and `last_appearance` properties have been added.

Doing the inserts for the `Crewman` objects with the properties `first_appearance` and `last_appearance` would have looked something like this:

```edgeql
insert Crewman {
  number := count(detached Crewman) +1,
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16),
};
```

But since we have a lot of `Crewman` objects already inserted, we can easily use the `update` and `set` syntax on all of them if we assume they all died at the same time (or if we don't care about being super precise). We'll do that in a second.

By the way, `cal::local_date` has a pretty simple YYYYMMDD format so casting from a string is pretty easy too:

```edgeql
select <cal::local_date>'1893-07-08';
```

But we imagined before that our dates and datetimes are being generated from a source that gives us individual numbers instead of strings, so we will continue to use that method.

Now we can update the `Crewman` objects. We'll give them all the same date to keep things simple:

```edgeql
update Crewman
set {
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16)
};
```

These dates will of course depend on our game. Can a `PC` actually visit the ship when it's sailing to England? Will there be missions to try to save the crew before Dracula kills them? If so, then we would need more precise dates and would need to make these properties into a `datetime`. But we're fine with these approximate dates for now.

## Adding defaults to a type, and the overloaded keyword

Now let's get back to inserting the new characters. First we'll insert Lucy:

```edgeql
insert NPC {
  name := 'Lucy Westenra',
  places_visited := (select City filter .name = 'London')
};
```

The other characters are from London too, which is the largest city in the world at the time. If we wanted to save ourselves some work, we could make London the default for `places_visited` for `NPC`. Let's give that a try. To do this we will need two things: `default` to declare a default, and the keyword `overloaded`. The word `overloaded` indicates that we are using `places_visited` in a different way than the `Person` type that we got it from.

Let's first see what error EdgeDB gives us if we forget the `overloaded` keyword. Try changing the `NPC` type to this and migrating the schema:

```sdl
type NPC extending Person {
property age -> HumanAge;
multi link places_visited -> Place {
  default := (select City filter .name = 'London');
  }
}
```

Impressive! It not only gives an error but tells us exactly what to do.

```
error: link 'places_visited' of object type 'default::NPC' must be declared using the `overloaded` keyword because it is defined in the following ancestor(s): default::Person
   ┌─ c:\rust\easy-edgedb\dbschema\default.esdl:27:3
   │
27 │ ╭   multi link places_visited -> Place {
28 │ │     default := (select City filter .name = 'London');
29 │ │   }
   │ ╰───^ error

edgedb error: cannot proceed until .esdl files are fixed
```

With `default` and `overloaded` added, it now looks like this:

```sdl
type NPC extending Person {
  property age -> HumanAge;
  overloaded multi link places_visited -> Place {
    default := (select City filter .name = 'London');
  }
}
```

## datetime_current()

One convenient function is {eql:func}` ``datetime_current()`` <docs:std::datetime_current>`, which gives the datetime right now. Let's try it out:

```edgeql-repl
db> select datetime_current();
{<datetime>'2023-05-28T10:18:56.889701Z'}
```

This can be useful if you want a post date when you insert an object. With this you can sort by date, delete the most recent item if you have a duplicate, and so on. Here is a quick example that creates three datetimes and then picks the most recent one using the `max()` function. Note that the third datetime created - the most recent - is the one returned by `max()`.

```edgeql-repl
db> with three_dates := {
  datetime_current(),
  datetime_current(),
  datetime_current()
  },
select three dates union max(three_dates);
{
  <datetime>'2023-05-28T10:26:55.744720Z',
  <datetime>'2023-05-28T10:26:55.744733Z',
  <datetime>'2023-05-28T10:26:55.744735Z',
  <datetime>'2023-05-28T10:26:55.744735Z',
}
```

Our game will have its `NPC` objects already in the database because they are all detailed in the book, but `PC` objects will only show up when a player decides to make a character. It could be useful to add a `date_created` property to the `PC` type so that we know when it was first made.

Let's imagine how it would look if we put it inside the `Place` type. This is close, but not quite:

```sdl
type PC extending Person {
  required property transport -> Transport;
  property created_at := datetime_current(); # this is new
}
```

Because `created_at` is a computable here, and computables are calculated when you *query* an object, this would generate the date when the query happens instead of when the object is inserted. So to make our `PC` objects have the date when you insert it, we can use `default` instead:

```sdl
type PC extending Person {
  required property transport -> Transport;
  property created_at -> datetime {
    default := datetime_current()
  }
}
```

Let's do a migration and give this a try with a second PC. We'll call him Max Demian and choose a `transport` but nothing else, but thanks to `created_at` having a default value we don't need to specify it ourselves. Let's `select` Max Demian right after his creation to see the date he was made.

```edgeql
with new_pc := (insert PC {
 name := 'Max Demian',
 transport := Transport.Train
 }),
 select new_pc {
 name,
 created_at
 };
```

The output will depend on when you do the insert, but it will look like this:

```
{default::PC {name: 'Max Demian', created_at: <datetime>'2023-05-30T01:13:28.022340Z'}}
```

## Using the 'for' and 'union' keywords

We're almost ready to insert our three new characters, and now we don't need to add `(select City filter .name = 'London')` every time. But wouldn't it be nice if we could use a single insert instead of three?

To do this, we can use a `for` loop, followed by the keyword `union`. First, here's the `for` part:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
```

In other words: take this set of three strings and do something to each one. `character_name` is the variable name we chose to call each string in this set.

`union` comes next, because it is the keyword used to join sets together. To understand why we need to write `union` in a `for` loop, take this example without `union`:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
select character_name ++ ' is great';
```

This generates an error because we are asking EdgeDB to give three query results at the same time. But what we want instead is a single query result (a single set) that contains the unified results of selecting each name. It will now work if we add `union` and surround `select` in parentheses, indicating that we are capturing the result of each `select` and joining them together:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
union (select character_name ++ ' is great');
```

With this, EdgeDB will select one name at a time, concatenate `' is great'` to the end, and unify them into a single set that it returns to us as follows:

```
{'John Seward is great', 'Quincey Morris is great', 'Arthur Holmwood is great'}
```

Now let's return to the `for` loop with the variable name `character_name`, which looks like this:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
union (
  insert NPC {
    name := character_name,
    lover := (select Person filter .name = 'Lucy Westenra'),
  }
);
```

We get three `uuid`s as a response to show that they were inserted.

Then we can check to make sure that it worked:

```edgeql
select NPC {
  name,
  places_visited: {
    name,
  },
  lover: {
    name,
  },
} filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'};
```

And as we hoped, they are all connected to Lucy now.

```
{
  default::NPC {
    name: 'John Seward',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
  default::NPC {
    name: 'Quincey Morris',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
  default::NPC {
    name: 'Arthur Holmwood',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
}
```

By the way, now we could use this method to insert our five `Crewman` objects inside one `insert` instead of doing it five times. Previously we used the `count()` function to insert their numbers, but we know that the book only ever has five crewmen so it will be easier to just use a `for` loop now. With this we can put their numbers inside a single set, and use the same `for` and `union` method to insert them. Of course, we already used `update` to change the inserts but from now on in our code their insert will look like this:

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

It's a good idea to familiarize yourself with {ref}`the order to follow <docs:ref_eql_statements_for>` when you use `for`:

```edgeql-synopsis
[ with with-item [, ...] ]

for variable in iterator-expr

union output-expr ;
```

The important part is the *iterator-expr* which needs to be a single simple expression that gives back some kind of set. Usually, it is a just a set inside `{` and `}`. It can also be a path, such as `NPC.places_visited`, or it can be a function call, such as `array_unpack()`. More complex expressions should have parentheses around them.

## An interesting migration

Now it's time to update Lucy with three lovers. Lucy has already ruined our plans to have `lover` as just a `link` (which means `single link`). We'll rename the `link lover` to `multi link lovers` instead and do a migration so that we can add all three of the men. This makes sense in any case, as other `Person` types could easily have more than one lover.

The migration output this time is a little interesting, as EdgeDB needed a bit of help to understand what we were trying to do. It first concluded that we were dropping the `lover` link, but after being told no, asked if we instead were trying to rename it.

```
c:\easy-edgedb>edgedb migration create
Connecting to an EdgeDB instance at localhost:10716...
did you drop link 'lover' of object type 'default::Person'? [y,n,l,c,b,s,q,?]
> n
did you rename link 'lover' of object type 'default::Person' to 'lovers'? [y,n,l,c,b,s,q,?]
> y
did you convert link 'lovers' of object type 'default::Person' to 'multi' cardinality? [y,n,l,c,b,s,q,?]
> y
```

Looking at the ddl output in the most recent `.edgeql` migration file in our `migrations` folder shows what commands were used to change the link:

```edgeql
ALTER TYPE default::Person {
      ALTER LINK lover {
          RENAME TO lovers;
      };
  };
  ALTER TYPE default::Person {
      ALTER LINK lovers {
          SET MULTI;
      };
  };
```

If we had said yes to dropping `link 'lover'`, then it would have created these commands instead:

```edgeql
ALTER TYPE default::Person {
    DROP LINK lover;
};
ALTER TYPE default::Person {
    CREATE MULTI LINK lovers: default::Person;
};
```

In this case the schema migration would still have worked, but the `lover` data would now be gone. Then we would have had to update the three men again to give them Lucy back:

```edgeql
update NPC filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
 set {
 lovers := (select Person filter .name = 'Lucy Westenra')
 };
```



Here is our update for her:

```edgeql
update NPC filter .name = 'Lucy Westenra'
set {
  lovers := (
    select Person filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};
```

Now we'll select her to make sure it worked. Let's use `like` this time for fun when doing the filter:

```edgeql
  select NPC {
    name,
    lovers: {
      name
    }
  } filter .name like 'Lucy%';
```

And this does indeed print her out with her three lovers.

```
{
  default::NPC {
    name: 'Lucy Westenra',
    lovers: {
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
    },
  },
}
```

## Overloading instead of making a new type

We can improve our schema a bit after having learned the `overloaded` keyword in this chapter. With this keyword we don't need the `HumanAge` type for `NPC` anymore. Right now it looks like this:

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

You will remember that we made this type because vampires can live forever, but humans only live up to 120. But now we can simplify things. First we move the `age` property over to the `Person` type. Then (inside the `NPC` type) we use `overloaded` to add a constraint on it there. Now `NPC` uses `overloaded` twice:

```sdl
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (select City filter .name = 'London');
  }
}
```

This is convenient because we can delete `age` from `Vampire` too. We don't need to `overload` here because vampires by default live forever the maximum value of 32767 for `int16` is, as far as we are concerned, forever. The type will now look like this:

```sdl
type Vampire extending Person {
  # property age -> int16; **Deleted now
  multi link slaves -> MinorVampire;
}
```

Okay, let's do the migration and then read the rest of the introduction for this chapter. It continues to explain what Lucy is up to:

> ...She chooses to marry Arthur Holmwood, and says sorry to the other two. The other two men are sad, but fortunately the three men become friends with each other. Dr. Seward is depressed and tries to concentrate on his work. He is a psychiatrist who works in an asylum close to a large mansion called Carfax not far outside London. Inside the asylum is a strange man named Renfield that Dr. Seward finds most interesting. Renfield is sometimes calm, sometimes completely crazy, and Dr. Seward doesn't know why he changes his mood so quickly. Also, Renfield seems to believe that he can get power from living things by eating them?! He's not a vampire, but seems to act similar sometimes.

Oops! Looks like Lucy doesn't have three lovers anymore.  We will have to remove her as a lover from the other two gentlemen. We'll just give them a sad empty set.

```edgeql
update NPC filter .name in {'John Seward', 'Quincey Morris'}
set {
  lovers := {} # 😢
};
```

That makes it easy to update Lucy's `lovers` link, since we know she now only shows up inside the `lovers` for Arthur Holmwood.

```edgeql
update NPC filter .name = 'Lucy Westenra'
set {
lovers := (
select Person filter NPC in .lovers
)};
```

Looks like we are mostly up to date now. The only thing left is to insert the mysterious Renfield. He is easy because he has no lover to `filter` for:

```edgeql
insert NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1893, 5, 26),
  strength := 10,
};
```

But he has some sort of relationship to Dracula, similar to the `MinorVampire` type but different. He is also quite strong (as we will see later), so we gave him a `strength` of 10. Later on we'll learn more and more about him and his relationship with Dracula.

[Here is all our code so far up to Chapter 9.](code.md)

<!-- quiz-start -->

## Time to practice

1. Why doesn't this insert work and how can it be fixed?

   ```edgeql
   for castle in ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
   union (
     insert Castle {
       name := castle
     }
   );
   ```

2. How would you do the same insert while displaying the castle's name at the same time?
3. How would you change the `Vampire` type if all vampires needed a minimum strength of 10?
4. How would you update all the `Person` types to show that they died on September 11, 1893?

   Hint: here's the type again:

   ```sdl
   abstract type Person {
     required property name -> str {
       delegated constraint exclusive;
     }
     property age -> int16;
     property strength -> int16;
     multi link places_visited -> Place;
     multi link lovers -> Person;
     property first_appearance -> cal::local_date;
     property last_appearance -> cal::local_date;
   }
   ```

5. All the `Person` characters that have an `e` or an `a` in their name have been brought back to life. How would you update to do this?

   Hint: "bringing back to life" means that `last_appearance` should return `{}`.

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Thick fog and a storm hit the city of Whitby._
