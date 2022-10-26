---
tags: Defaults, Overloading, For Loops
---

# Chapter 9 - Strange events in England

For this chapter we've gone back in time a few weeks to when the ship left Varna and Mina and Lucy haven't left for Whitby yet. The introduction is also split into two parts. Here's the first:

> We still don't know where Jonathan is, and the ship The Demeter is on its way to England with Dracula inside. Meanwhile, Mina Harker is in London writing letters to her friend Lucy Westenra. Lucy has three boyfriends (named Dr. John Seward, Quincey Morris, and Arthur Holmwood), and they all want to marry her....

## Working with dates some more

It looks like we have some more people to insert. But first, let's think about the ship a little more. Everyone on the ship was killed by Dracula, but we don't want to delete the crew because they are still part of our game. The book tells us that the ship left on the 6th of July, and the last person (the captain) died on the 4th of August (in 1887).

This is a good time to add two new properties to the `Person` type to indicate when a character is present. We'll call them `first_appearance` and `last_appearance`. The name `last_appearance` is a bit better than `death`, because for the game it doesn't matter: we just want to know when characters are there or not.

For these two properties we will just use `cal::local_date` for the sake of simplicity. There is also `cal::local_datetime` that includes time, but we should be fine with just the date. (And of course there is the `cal::local_time` type with just the time of day that we have in our `Date` type.)

Doing an insert for the `Crewman` objects with the properties `first_appearance` and `last_appearance` will now look something like this:

```edgeql
INSERT Crewman {
  number := count(DETACHED Crewman) +1,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
};
```

And since we have a lot of `Crewman` objects already inserted, we can easily use the `UPDATE` and `SET` syntax on all of them if we assume they all died at the same time (or if being super precise doesn't matter).

Since `cal::local_date` has a pretty simple YYYYMMDD format, the easiest way to use it in an insert would be just casting from a string:

```edgeql
SELECT <cal::local_date>'1887-07-08';
```

But we imagined before that we had a function that gives separate numbers to put into a function, so we will continue to use that method.

Before we used the function `std::to_datetime` which took seven parameters; this time we'll use a similar but shorter {eql:func}`docs:cal::to_local_date` function. It just takes three integers.

Here are its signatures (we're using the third):

```
cal::to_local_date(s: str, fmt: OPTIONAL str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

Now we update the `Crewman` objects and give them all the same date to keep things simple:

```edgeql
UPDATE Crewman
SET {
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16)
};
```

This will of course depend on our game. Can a `PC` actually visit the ship when it's sailing to England? Will there be missions to try to save the crew before Dracula kills them? If so, then we will need more precise dates. But we're fine with these approximate dates for now.

## Adding defaults to a type, and the overloaded keyword

Now let's get back to inserting the new characters. First we'll insert Lucy:

```edgeql
INSERT NPC {
  name := 'Lucy Westenra',
  places_visited := (SELECT City FILTER .name = 'London')
};
```

Hmm, it looks like we're doing a lot of work to insert 'London' every time we add a character. We have three characters left and they will all be from London too. To save ourselves some work, we can make London the default for `places_visited` for `NPC`. To do this we will need two things: `default` to declare a default, and the keyword `overloaded`. The word `overloaded` indicates that we are using `places_visited` in a different way than the `Person` type that we got it from.

With `default` and `overloaded` added, it now looks like this:

```sdl
type NPC extending Person {
  property age -> HumanAge;
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

## datetime_current()

One convenient function is {eql:func}` ``datetime_current()`` <docs:std::datetime_current>`, which gives the datetime right now. Let's try it out:

```edgeql-repl
edgedb> SELECT datetime_current();
{<datetime>'2020-11-17T06:13:24.418765000Z'}
```

This can be useful if you want a post date when you insert an object. With this you can sort by date, delete the most recent item if you have a duplicate, and so on. Let's imagine how it would look if we put it inside the `Place` type. This is close, but not quite:

```sdl
abstract type Place {
  required property name -> str {
    constraint exclusive;
  }
  property modern_name -> str;
  property important_places -> array<str>;
  property post_date := datetime_current(); # this is new
}
```

This will actually generate the date when you *query* a `Place` object, not when you insert it. So to make a `Place` type that would have the date when you insert it, we can use `default` instead:

```sdl
abstract type Place {
  required property name -> str {
    constraint exclusive;
  }
  property modern_name -> str;
  property important_places -> array<str>;
  property post_date -> datetime {
    default := datetime_current()
  }
}
```

We don't need this in our schema so we won't change `Place`, but this is how you would do it.

## Using FOR and UNION

We're almost ready to insert our three new characters, and now we don't need to add `(SELECT City FILTER .name = 'London')` every time. But wouldn't it be nice if we could use a single insert instead of three?

To do this, we can use a `FOR` loop, followed by the keyword `UNION`. First, here's the `FOR` part:

```edgeql
FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
```

In other words: take this set of three strings and do something to each one. `character_name` is the variable name we chose to call each string in this set.

`UNION` comes next, because it is the keyword used to join sets together. For example, this query:

```edgeql
WITH city_names := (SELECT City.name),
  castle_names := (SELECT Castle.name),
SELECT city_names UNION castle_names;
```

joins the names together to give us the output `{'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Castle Dracula'}`.

Now let's return to the `FOR` loop with the variable name `character_name`, which looks like this:

```edgeql
FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
UNION (
  INSERT NPC {
    name := character_name,
    lover := (SELECT Person FILTER .name = 'Lucy Westenra'),
  }
);
```

We get three `uuid`s as a response to show that they were entered.

Then we can check to make sure that it worked:

```edgeql
SELECT NPC {
  name,
  places_visited: {
    name,
  },
  lover: {
    name,
  },
} FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'};
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

By the way, now we could use this method to insert our five `Crewman` objects inside one `INSERT` instead of doing it five times. We can put their numbers inside a single set, and use the same `FOR` and `UNION` method to insert them. Of course, we already used `UPDATE` to change the inserts but from now on in our code their insert will look like this:

```edgeql
FOR n IN {1, 2, 3, 4, 5}
UNION (
  INSERT Crewman {
    number := n,
    first_appearance := cal::to_local_date(1887, 7, 6),
    last_appearance := cal::to_local_date(1887, 7, 16),
  }
);
```

It's a good idea to familiarize yourself with {ref}`the order to follow <docs:ref_eql_statements_for>` when you use `FOR`:

```edgeql-synopsis
[ WITH with-item [, ...] ]

FOR variable IN iterator-expr

UNION output-expr ;
```

The important part is the *iterator-expr* which needs to be a single simple expression that gives back some kind of set. Usually, it is a just a set inside `{` and `}`. It can also be a path, such as `NPC.places_visited`, or it can be a function call, such as `array_unpack()`. More complex expressions should have parentheses around them.

Now it's time to update Lucy with three lovers. Lucy has already ruined our plans to have `lover` as just a `link` (which means `single link`). We'll set it to `multi link` instead so we can add all three of the men. Here is our update for her:

```edgeql
UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (
    SELECT Person FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};
```

Now we'll select her to make sure it worked. Let's use `LIKE` this time for fun when doing the filter:

```edgeql
SELECT NPC {
  name,
  lover: {
    name
  }
} FILTER .name LIKE 'Lucy%';
```

And this does indeed print her out with her three lovers.

```
{
  default::NPC {
    name: 'Lucy Westenra',
    lover: {
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
    },
  },
}
```

## Overloading instead of making a new type

So now that we know the keyword `overloaded`, we don't need the `HumanAge` type for `NPC` anymore. Right now it looks like this:

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
    default := (SELECT City filter .name = 'London');
  }
}
```

This is convenient because we can delete `age` from `Vampire` too:

```sdl
type Vampire extending Person {
  # property age -> int16; **Delete this one now**
  multi link slaves -> MinorVampire;
}
```

You can see that a good usage of abstract types and the `overloaded` keyword lets you simplify your schema if you do it right.

Okay, let's read the rest of the introduction for this chapter. It continues to explain what Lucy is up to:

> ...She chooses to marry Arthur Holmwood, and says sorry to the other two. The other two men are sad, but fortunately the three men become friends with each other. Dr. Seward is depressed and tries to concentrate on his work. He is a psychiatrist who works in an asylum close to a large mansion called Carfax not far outside London. Inside the asylum is a strange man named Renfield that Dr. Seward finds most interesting. Renfield is sometimes calm, sometimes completely crazy, and Dr. Seward doesn't know why he changes his mood so quickly. Also, Renfield seems to believe that he can get power from living things by eating them. He's not a vampire, but seems to act similar sometimes.

Oops! Looks like Lucy doesn't have three lovers anymore. Now we'll have to update her to only have Arthur:

```edgeql
UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (SELECT DETACHED NPC FILTER .name = 'Arthur Holmwood'),
};
```

And then remove her from the other two. We'll just give them a sad empty set.

```edgeql
UPDATE NPC FILTER .name in {'John Seward', 'Quincey Morris'}
SET {
  lover := {} # 😢
};
```

Looks like we are mostly up to date now. The only thing left is to insert the mysterious Renfield. He is easy because he has no lover to `FILTER` for:

```edgeql
INSERT NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1887, 5, 26),
  strength := 10,
};
```

But he has some sort of relationship to Dracula, similar to the `MinorVampire` type but different. He is also quite strong (as we will see later), so we gave him a `strength` of 10. Later on we'll learn more and more about him and his relationship with Dracula.

[Here is all our code so far up to Chapter 9.](code.md)

<!-- quiz-start -->

## Time to practice

1. Why doesn't this insert work and how can it be fixed?

   ```edgeql
   FOR castle IN ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
   UNION (
     INSERT Castle {
       name := castle
     }
   );
   ```

2. How would you do the same insert while displaying the castle's name at the same time?
3. How would you change the `Vampire` type if all vampires needed a minimum strength of 10?
4. How would you update all the `Person` types to show that they died on September 11, 1887?

   Hint: here's the type again:

   ```sdl
   abstract type Person {
     required property name -> str {
       constraint exclusive;
     }
     property age -> int16;
     property strength -> int16;
     multi link places_visited -> Place;
     multi link lover -> Person;
     property first_appearance -> cal::local_date;
     property last_appearance -> cal::local_date;
   }
   ```

5. All the `Person` characters that have an `e` or an `a` in their name have been brought back to life. How would you update to do this?

   Hint: "bringing back to life" means that `last_appearance` should return `{}`.

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Thick fog and a storm hit the city of Whitby._
