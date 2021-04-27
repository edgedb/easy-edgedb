---
tags: Expression On, Error Messages
---

# Chapter 15 - Time to start vampire hunting

> It's good that Jonathan is back, but he is still in shock. He doesn't know if the experience with Dracula was real or not, and thinks he might be crazy. But then he meets Van Helsing who tells him that it was all true. Jonathan hears this and becomes strong and confident again. Now they begin to search for Dracula. The others learn that the Carfax mansion across from Dr. Seward's asylum is the one that Dracula bought. So that's why Renfield was so strongly affected... They search the house when the sun is up and find boxes of earth in which Dracula sleeps. They destroy them all in Carfax, but there are still many left in London. If they don't destroy the other boxes, Dracula will be able to rest in them during the day and terrorize London every night when the sun goes down.

## More abstract types

This chapter they learned something about vampires: they need to sleep in coffins (boxes for dead people) with holy earth during the day. That's why Dracula brought 50 of them over by ship on the Demeter. This is important for the mechanics of our game so we should create a type for this. And if we think about it:

- Each place in the world either has coffins or doesn't have them,
- Has coffins = vampires can enter and terrorize the people,
- If a place has coffins, we should know how many of them there are.

This sounds like a good case for an abstract type. Here it is:

```sdl
abstract type HasCoffins {
  required property coffins -> int16 {
    default := 0;
  }
}
```

Most places will not have a special vampire coffin, so the default is 0. The `coffins` property is just an `int16`, and vampires can remain close to a place if the number is 1 or greater. In the mechanics of our game we would probably give vampires an activity radius of about 100 km from a place with a coffin. That's because of the typical vampire schedule which is usually as follows:

- Wake up refreshed in the coffin after the sun goes down, get ready to leave by 8 pm to find people to terrorize
- Feel free because the night has just begun, start moving away from the safety of the coffins to find victims. May use a horse-driven carriage at 25 kph to do so.
- Around 1 or 2 pm, start to feel nervous. The sun will be up in about 5 hours. Is there enough time to get home?

So the part between 8 pm and 1 am is when the vampire is free to move away, and at 25 kph we get an activity radius of about 100 km around a coffin. At that distance, even the bravest vampire will start running back towards home by 2 am.

With a more complex game we could imagine that vampire terrorism is worse in the winter (activity radius = about 150 km), but we won't get into those details.

With our abstract type done, we will want to have a lot of types `extending` this. First we can have `Place` extend it, and that gives it to all the other location types such as `City`, `OtherPlace`, etc.:

```sdl
abstract type Place extending HasCoffins {
  required property name -> str {
    constraint exclusive;
  };
  property modern_name -> str;
  property important_places -> array<str>;
}
```

Ships are also big enough to have coffins (the Demeter had 50 of them, after all) so we'll extend for `Ship` as well:

```sdl
type Ship extending HasCoffins {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

If we want, we can now make a quick function to test whether a vampire can enter a place:

```sdl
function can_enter(person_name: str, place: HasCoffins) -> str
  using (
    with vampire := (SELECT Person filter .name = person_name LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

You'll notice that `person_name` in this function actually just takes a string that it uses to select a `Person`. So technically it could say something like 'Jonathan Harker cannot enter'. It has a `LIMIT 1` though, and we can probably trust that the user of the function will use it properly. If we can't trust the user of the function, there are some options:

- Overload the function to have two signatures, one for each type of Vampire:

```sdl
function can_enter(vampire: Vampire, place: HasCoffins) -> str
function can_enter(vampire: MinorVampire, place: HasCoffins) -> str
```

- Create an abstract type (like `type IsVampire`) and extend it for `Vampire` and `MinorVampire`. Then `can_enter` can have this signature: `function can_enter(vampire: IsVampire, place: HasCoffins) -> str`

Overloading the function is probably the easier option, because we wouldn't need to do an explicit migration.

One other area where you need to trust the user of the function is seen in the return type, which is just `-> str`. Beyond just returning a string, this return type also means that the function won't be called if the input is empty. So what if you want it to be called anyway? If you want it to be called no matter what, you can change the return type to `-> OPTIONAL str`. [The documentation](https://www.edgedb.com/docs/edgeql/overview#optional) explains it like this: `the function is called normally when the corresponding argument is empty`. And: `A notable example of a function that gets called on empty input is the coalescing operator.`

Interesting! You'll remember the coalescing operator `?` that we first saw in Chapter 12. And when we look at [its signature](https://www.edgedb.com/docs/edgeql/funcops/set/#operator::COALESCE), you can see the `OPTIONAL` in there:

`OPTIONAL anytype ?? SET OF anytype -> SET OF anytype`

So those are some ideas for how to set up your functions depending on how you think people might use them.

Now let's give London some coffins. According to the book, our heroes destroyed 29 coffins at Carfax that night, which leaves 21 in London.

```edgeql
UPDATE City filter .name = 'London'
SET {
  coffins := 21
};
```

Now we can finally call up our function and see if it works:

```edgeql
SELECT can_enter('Count Dracula', (SELECT City filter .name = 'London'));
```

We get `{'Count Dracula can enter.'}`.

Some other possible ideas for improvement later on for `can_enter()` are:

- Move the property `name` from `Place` and `Ship` over to `HasCoffins`. Then the user could just enter a string. The function would then use it to `SELECT` the type and then display its name, giving a result like "Count Dracula can enter London."
- Require a date in the function so that we can check if the vampire is dead or not first. For example, if we entered a date after Lucy died, it would just display something like `vampire.name ++ ' is already dead on ' ++ <str>.date ++ ' and cannot enter ' ++ city.name`.

## More constraints

Let's look at some more constraints. We've seen `exclusive` and `max_value` already, but there are [some others](https://www.edgedb.com/docs/datamodel/constraints) that we can use as well.

There is one called `max_len_value` that makes sure that a string doesn't go over a certain length. That could be good for our `PC` type, which we created many chapters ago. We only used it once as a test, because we don't have any players yet. We are still just using the book to populate the database with `NPC`s for our imaginary game. `NPC`s won't need this constraint because their names are already decided, but `max_len_value()` is good for `PC`s to make sure that players don't choose crazy names. We'll change it to look like this:

```sdl
type PC extending Person {
  required property transport -> Transport;
  overloaded required property name -> str {
    constraint max_len_value(30);
  }
}
```

Then when we try to insert a `PC` with a name that is too long, it will refuse with `ERROR: ConstraintViolationError: name must be no longer than 30 characters.`

Another convenient constraint is called `one_of`, and is sort of like an enum. One place in our schema where we could use it is `property title -> str;` in our `Person` type. You'll remember that we added that in case we wanted to generate names from various parts (first name, last name, title, degree...). This constraint could work to make sure that people don't just make up their own titles:

```sdl
property title -> str {
  constraint one_of('Mr.', 'Mrs.', 'Ms.', 'Lord')
}
```

For us it's probably not worth it to add a `one_of` constraint though, as there are probably too many titles throughout the book (Count, German _Herr_, etc. etc.).

Another place you could imagine using a `one_of` is in the months, because the book only goes from May to October of the same year. If we had an object type generating a date then you could have this sort of constraint inside it:

```sdl
property month -> int64 {
  constraint one_of(5, 6, 7, 8, 9, 10)
}
```

But that will depend on how the game works.

Now let's learn about perhaps the most interesting constraint in EdgeDB:

## expression on: the most flexible constraint

One particularly flexible constraint is called [`expression on`](https://www.edgedb.com/docs/datamodel/constraints#constraint::std::expression), which lets us add any expression we want. After `expression on` you add the expression (in brackets) that must be true to create the type. In other words: "Create this type _as long as_ (insert expression here)".

Let's say we need a type `Lord` for some reason later on, and all `Lord` types must have the word 'Lord' in their name. We can constrain the type to make sure that this is always the case. For this, we will use a function called [contains()](https://www.edgedb.com/docs/edgeql/funcops/generic#function::std::contains) that looks like this:

```sdl
std::contains(haystack: str, needle: str) -> bool
```

It returns `{true}` if the `haystack` (a string) contains the `needle` (usually a shorter string).

We can write the constraint with `expression on` and `contains()` like this:

```sdl
type Lord extending Person {
  constraint expression on (
    contains(__subject__.name, 'Lord') = true
  );
}
```

`__subject__` there refers to the type itself.

Now when we try to insert a `Lord` without it, it won't work:

```edgeql
INSERT Lord {
  name := 'Billy'
  # Other stuff..
};
```

But if the `name` is 'Lord Billy' (or 'Lord' anything), it will work.

While we're at it, let's practice doing a `SELECT` and `INSERT` at the same time so we see the output of our `INSERT` right away. We'll change `Billy` to `Lord Billy` and say that Lord Billy (considering his great wealth) has visited every place in our database.

```edgeql
SELECT (
  INSERT Lord {
    name := 'Lord Billy',
    places_visited := (SELECT Place),
  }
) {
  name,
  places_visited: {
    name
  }
};
```

Now that `.name` contains the substring `Lord`, it works like a charm:

```
{
  default::Lord {
    name: 'Lord Billy',
    places_visited: {
      default::Castle {name: 'Castle Dracula'},
      default::City {name: 'Whitby'},
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
      default::City {name: 'London'},
      default::Country {name: 'Romania'},
      default::Country {name: 'Slovakia'},
    },
  },
}
```

## Setting your own error messages

Since `expression on` is so flexible, you can use it in almost any way you can imagine. But it's not certain that the user will know about this constraint - there's no message informing the user of this. Meanwhile, the automatically generated error message we have right now is not helping the user at all:

`ERROR: ConstraintViolationError: invalid Lord`

So there's no way to tell that the problem is that `name` needs `'Lord'` inside it. Fortunately, constraints allow you to set your own error message just by using `errmessage`, like this: `errmessage := "All lords need 'Lord' in their name."`

Now the error becomes:

`ERROR: ConstraintViolationError: All lords need 'Lord' in their name.`

Here's the `Lord` type now:

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord') = true) {
    errmessage := "All lords need \'Lord\' in their name";
  };
};
```

## Links in two directions

Back in Chapter 6 we removed `link master` from `MinorVampire`, because `Vampire` already has `link slaves` to the `MinorVampire` type. One reason was complexity, and the other was because `DELETE` becomes impossible because they both depend on each other. But now that we know how to use reverse links, we can put `master` back in `MinorVampire` if we want.

(Note: we won't actually change the `MinorVampire` type here because we already know how to access `Vampire` with a reverse lookup, but this is how to do it)

First, here is the `MinorVampire` type at present:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
}
```

To add the `master` link again, the best way is to start with a property called `master_name` that is just a string. Then we can use this in a reverse search to link to the `Vampire` type if the name matches. It's a single link, so we'll add `LIMIT 1` (it won't work otherwise). Here is what the type would look like now:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
  link master := (SELECT .<slaves[IS Vampire] LIMIT 1);
  required single property master_name -> str;
};
```

Now let's test it out. We'll make a vampire named Kain, who has two `MinorVampire` slaves named Billy and Bob.

```edgeql
INSERT Vampire {
  name := 'Kain',
  slaves := {
    (INSERT MinorVampire {
      name := 'Billy',
      master_name := 'Kain'
    }),
    (INSERT MinorVampire {
      name := 'Bob',
      master_name := 'Kain'
    })
  }
};
```

Now if the `MinorVampire` type works as it should, we should be able to see Kain via `link master` inside `MinorVampire` and we won't have to do a reverse lookup. Let's check:

```edgeql
SELECT MinorVampire {
  name,
  master_name,
  master: {
    name
  }
} FILTER .name IN {'Billy', 'Bob'};
```

And the result:

```
{
  default::MinorVampire {
    name: 'Billy',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
  default::MinorVampire {
    name: 'Bob',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
}
```

Beautiful! All the information is right there.

[Here is all our code so far up to Chapter 15.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you create a type called Horse with a `required property name -> str` that can only be 'Horse'?

2. How would you let the user know that it needs to be called 'Horse'?

3. How would you make sure that `name` for type `PC` is always between 5 and 30 characters in length?

   Try it first with `expression on`.

4. How would you make a function called `display_coffins` that pulls up all the `HasCoffins` types with more than 0 coffins?

5. How would you make it without touching the schema?

[See the answers here.](answers.md)

Up next in Chapter 16: [Could Renfield be of help?](../chapter16/index.md)

<!-- quiz-end -->
