---
tags: Defaults, Overloading, For Loops
---

# Chapter 9 - Strange events in England

The episode for this chapter is a flashback to see why everybody is in the town of Whitby in the first place. Our characters are from London, after all, not a small town on the east coast of England. We'll go back in time a few weeks to when the ship carrying Dracula left Varna, when Mina and Lucy were still back home. The introduction is also split into two parts. Here's the first:

> We still don't know where Jonathan is, and the ship The Demeter is on its way to England with Dracula inside. Meanwhile, Mina Harker is in London writing letters to her friend Lucy Westenra. Lucy has three boyfriends (named Dr. John Seward, Quincey Morris, and Arthur Holmwood), and they all want to marry her....

## Working with dates some more

It looks like we have some more people to insert, starting with Lucy:

```edgeql
insert NPC {
  name := 'Lucy Westenra',
  places_visited := (select City filter .name = 'London')
};
```

But let's think about the ship a little more before we insert the rest. Everyone on the ship was killed by Dracula, but that doesn't mean that we want to delete them from the database. After all, the crewmembers on the ship were alive before and could have interacted with some `PC` objects during that time. The book tells us that the ship left on the 6th of July, and the last person (the captain) died on the 4th of August.

This is a good time to add two new properties to the `Person` type to indicate when a character is alive and present in the game. We'll call these properties `first_appearance` and `last_appearance`. The name `last_appearance` is a bit better than `death`, because for the game it doesn't matter: we just want to know when characters are there or not.

For these two properties we will just use `cal::local_date` for the sake of simplicity. There is also a `cal::local_datetime` type that includes the time, but we should be fine with just the date.

We used the `std::to_datetime` function before which took seven parameters to make a `DateTime`, and `cal::local_date` has a similar but but shorter function called {eql:func}`docs:cal::to_local_date`. It just takes three integers.

Here are all of its signatures (we're using the third one):

```
cal::to_local_date(s: str, fmt: optional str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

In addition, `cal::local_date` has a pretty simple YYYYMMDD format so casting from a string is pretty easy too:

```edgeql
select <cal::local_date>'1893-07-08';
```

But we decided before that the dates and datetimes in our game are being generated from a source that gives us individual numbers instead of strings, so we will continue to use that method.

So let's do a migration now that these `first_appearance` and `last_appearance` properties have been added.

Doing the inserts for the `Crewman` objects with the properties `first_appearance` and `last_appearance` would have looked something like this:

```edgeql
insert Crewman {
  number := count(detached Crewman) +1,
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16),
};
```

But since we already have the `Crewman` objects in the database, we can easily use the `update` and `set` syntax on all of them if we assume they all died at the same time (or if we don't care about being super precise). The query to update them looks like this:

```edgeql
update Crewman
set {
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance  := cal::to_local_date(1893, 7, 16)
};
```

The date type to choose will of course depend on our game. Can a `PC` actually visit the ship when it's sailing to England? Will there be missions to try to save the crew before Dracula kills them? If so, then we would need more precision and a `cal::local_datetime` would make more sense. But we're fine with these approximate dates for now.

## datetime_current() and datetime_of_statement()

Speaking of dates and datetimes, sometimes it can be useful to know the datetime right now. EdgeDB has two convenient functions for this: {eql:func}` ``datetime_current()`` <docs:std::datetime_current>` and {eql:func}` ``datetime_of_statement()`` <docs:std::datetime_of_statement>`. Let's try the first one out and see what it returns:

```
db> select datetime_current();
{<datetime>'2023-05-28T10:18:56.889701Z'}
```

This can be useful if you want a post date when you insert an object. With this you can sort objects by date, delete the most recent item if you have a duplicate, and so on.

Note though that `datetime_current()` will not return the exact same date as another call to `datetime_current()` inside the same statement. This is because `datetime_current()` returns the datetime at which the _function is called_, not the datetime of the statement that it's in.

We can easily see this by calling the function twice. The timestamp is almost the same, but not quite: one is a few microseconds after the other.

```
db> select (datetime_current(), datetime_current());
{
  (<datetime>'2023-07-16T10:55:55.498380Z', 
   <datetime>'2023-07-16T10:55:55.498394Z')
}
```

However, if we change the function to `datetime_of_statement()`, then the exact same datetime will be returned no matter how many times we call it:

```
edgedb> select (datetime_of_statement(), datetime_of_statement());
{
  (<datetime>'2023-07-16T10:58:03.384721Z', 
   <datetime>'2023-07-16T10:58:03.384721Z')
}
```

Or even simpler, a query to see if one function output equals the other.

```
select (
  datetime_of_statement() = datetime_of_statement(),
  datetime_current() = datetime_current(), 
);
```

This should return `true` followed by `false` - unless EdgeDB was fast enough to call `datetime_current()` twice in the same microsecond.

We can put one of these functions to use in our schema too. Our game will have its `NPC` objects already in the database because they are all detailed in the book, and they will be in the database long before anybody begins playing our game. However, `PC` objects will only show up when a player decides to make a character. It could be useful to add a `date_created` property to the `PC` type so that we know when it was first made.

Let's imagine how it would look if we put it inside the `Person` type. This is close, but not quite:

```sdl
type PC extending Person {
  required class: Class;
  created_at := datetime_of_statement(); # this is new
}
```

Because `created_at` is a computable here, and computables are calculated when you *query* an object, this would generate the date when the query happens instead of when the object is inserted. It would effectively be a `datetime_of_the_query_you_just_made` property. So to make our `PC` objects have the date when you first inserted it, we can use `default` instead. And since we are adding a default value, we might as well make `created_at` a `required` property.

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement()
  }
}
```

Let's do a migration and give this a try with a second PC. We'll call him Max Demian and choose a `class` but nothing else, but thanks to `created_at` having a default value we don't need to specify it ourselves. Let's `select` Max Demian right after his creation to see the date he was made.

```edgeql
with new_pc := (insert PC {
   name := 'Max Demian',
   class := Class.Mystic
 }),
 select new_pc {
   name,
   created_at
 };
```

The output will depend on when you do the insert, but will look something like this:

```
{default::PC {name: 'Max Demian', 
created_at: <datetime>'2023-05-30T01:13:28.022340Z'}}
```

## Using the 'for' and 'union' keywords

We're almost ready to insert our three new characters. They are all `NPC`s that have been to London before, so the only difference between them at the moment is their name. Wouldn't it be nice if we could use a single insert instead of three?

To do this, we can use a `for` loop, followed by the keyword `union`. First, here's the `for` part:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
```

In other words: take this set of three strings and do something with each one. `character_name` is the variable name we chose to call each string in this set.

`union` comes next, because it is the keyword used to join sets together. To understand why we need to write `union` in a `for` loop, take this example without `union`:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
select character_name ++ ' is great';
```

This generates an error because we are asking EdgeDB to make three queries when we are actually trying to do one. But what we want instead is a single query result (a single set) that contains the *unified* results of selecting each name. That's what the `union` keyword is for here, as it will join the sets together for us. We will also have to surround `select` in parentheses, indicating that we are capturing the result of each `select` and joining them together. Altogether the query now looks like this:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
union (select character_name ++ ' is great');
```

With this EdgeDB will select one name at a time, concatenate `' is great'` on the end, and unify them into a single set that it returns to us as follows:

```
{'John Seward is great', 'Quincey Morris is great', 'Arthur Holmwood is great'}
```

Now let's use what we learned to insert the three characters at once! This query will start with the three character names, but now each time we will insert an `NPC` with its name:

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
union (
  insert NPC {
    name := character_name,
    places_visited := (select City filter .name = 'London'),
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

By the way, now we could also use the `for` keyword to insert our five `Crewman` objects inside one `insert` instead of doing it five times. Previously we used the `count()` function to insert their numbers, but we know that the book only ever has five crewmen so it will be easier to just use a `for` loop now. With this we can put their numbers inside a single set, and use the same `for` and `union` method to insert them. Of course, we already used `update` to change the inserts but from now on in our code their insert will look like this:

```edgeql
for n in {1, 2, 3, 4, 5}
union (
  insert Crewman {
    number := n,
    first_appearance := cal::to_local_date(1893, 7, 6),
    last_appearance  := cal::to_local_date(1893, 7, 16),
  }
);
```

It's a good idea to familiarize yourself with {ref}`the order to follow <docs:ref_eql_statements_for>` when you use `for`. Here is the syntax information from the documentation on `for`:

```edgeql-synopsis
[ with with-item [, ...] ]

for variable in iterator-expr

union output-expr ;
```

The important part is the *iterator-expr* which needs to be a single simple expression that gives back some kind of set. Usually, it is a just a set inside `{` and `}`. It can also be a path, such as `NPC.places_visited`, or it can be a function call, such as `array_unpack()`. More complex expressions should have parentheses around them.

## An interesting migration

Now it's time to update Lucy with three lovers. Lucy has already ruined our plans to have `lover` as just a single link. We'll rename the `lover` link to `multi lovers` to make it a multi link instead and do a migration so that she can be linked to all three of the men. This change makes sense in any case, as other `Person` types could easily have more than one lover.

Sometimes migration output can be a little interesting, as EdgeDB doesn't always exactly know what we are trying to do. For example, during this migration via a previous version of EdgeDB in the summer of 2023, EdgeDB first concluded that we were dropping the `lover` link, but after being told no, asked if we instead were trying to rename it. This is a good reminder to pay attention to the questions when doing a migration, because (though rare) EdgeDB will sometimes misunderstand what you are trying to do. Here is the migration output from that time:

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

In a case such as this you might feel the need to make sure that EdgeDB has understood what you have asked it to do before typing `edgedb migrate`. Looking at the ddl output in the most recent `.edgeql` migration file in our `migrations` folder shows what commands were used to change the link:

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

That looks correct!

If, on the other hand, we had said yes to dropping `link 'lover'`, then we would have seen these commands instead:

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
update NPC filter .name in 
 {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
   set {
    lovers := (select Person filter .name = 'Lucy Westenra')
 };
```

Here is our update for her:

```edgeql
update NPC filter .name = 'Lucy Westenra'
set {
  lovers := (
    select Person filter .name in
      {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
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

And this does indeed return Lucy with a link to her three lovers.

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

## An uninteresting migration

This migration was pretty interesting, but what if you are doing a really boring migration that you are sure is safe to do? You might have pasted a schema that you've used before, or could have deleted everything inside a module, or anything else that makes you wish you didn't have to reply to every question from the CLI.

In this case there is a solution: just add `--non-interactive` after `edgedb migration create`, and the CLI will be quiet and put your migration together — if it can. If there are any changes that do require your input, such as making a parameter `required`, you will instead see the following output: `edgedb error: Cannot apply migration without user input`.

Note that a non-interactive migration still only creates a migration script, so it won't apply the migration before you have had a chance to review the script yourself. So don't worry!

## Overloading instead of making a new type

Last chapter we learned the `overloaded` keyword, and we can use it now to improve our schema a bit. Remember the `HumanAge` scalar type we created before? Right now it looks like this:

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

We originally made `HumanAge` for humans to use because vampires can live forever, but humans only live up to 120. But now we can simplify things. First we move the `age` property over to the `Person` type. Then (inside the `NPC` type) we use `overloaded` to add a constraint on it there:

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120);
  }
}
```

This is convenient because we can delete `age` from `Vampire` too. We don't need to use `overloaded` here because vampires can live up to the maximum value of 32767 for `int16` which, as far as we are concerned, is forever. The `Vampire` type will now look like this:

```sdl
type Vampire extending Person {
  # age: int16; **Deleted now
  multi slaves: MinorVampire;
}
```

Okay, let's do another migration and then read the rest of the introduction for this chapter. It continues to explain what Lucy is up to:

> ...Lucy chooses to marry Arthur Holmwood, and says sorry to the other two. The other two men are sad, but fortunately all three men become friends with each other.
>
> Dr. Seward is depressed and tries to concentrate on his work. He is a psychiatrist who works in an asylum close to a large mansion called Carfax not far outside London. Inside the asylum is a strange man named Renfield that Dr. Seward finds most interesting. Renfield is sometimes calm, sometimes completely crazy, and Dr. Seward doesn't know why he changes his mood so quickly. Also, Renfield seems to believe that he can get power from living things by eating them!! Renfield isn't exactly a vampire, but seems to act similar sometimes.

Oops! Looks like Lucy doesn't have three lovers anymore.  We will have to remove her as a lover from the other two gentlemen. We'll just update each of them a sad empty set.

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
   for castle in
     ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
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
     required name: str {
       delegated constraint exclusive;
     }
     age: int16;
     strength: int16;
     multi places_visited: Place;
     multi lovers: Person;
     first_appearance: cal::local_date;
     last_appearance: cal::local_date;
   }
   ```

5. All the `Person` characters that have an `e` or an `a` in their name have been brought back to life. How would you update to do this?

   Hint: "bringing back to life" means that `last_appearance` should return `{}`.

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Thick fog and a storm hit the city of Whitby._
