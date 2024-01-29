---
tags: Expression On, Error Messages
---

# Chapter 15 - The vampire hunt begins

> It's good that Jonathan is back, but he is still in shock. He doesn't know if the experience with Dracula was real or not, and thinks he might be crazy. But then Jonathan meets Van Helsing who tells him that it was all true. Now that he knows everything was true, Jonathan becomes strong and confident again.
>
> Our heroes begin the search for Dracula. The others learn that the Carfax mansion across from Dr. Seward's asylum is the one that Dracula bought. So that's why Renfield was so strongly affected!
>
> The heroes search the house when the sun is up and find boxes of earth in which Dracula sleeps. They destroy them all in Carfax, but there are still many left in London. If they don't destroy the other boxes, Dracula will be able to rest in them during the day and terrorize London every night when the sun goes down.

## More abstract types

Our heroes learned something about vampires in this chapter: vampires need to sleep in coffins (boxes for dead people) with holy earth during the day. That's why Dracula brought 50 of them over by ship on the Demeter. This is important for the mechanics of our game so we should create a type for this. And if we think about it:

- Each place in the world either has coffins or doesn't have them,
- A place that has coffins is a place that vampires can enter and terrorize the people,
- If a place has coffins, we should know how many of them there are.

This sounds like a good case for an abstract type. Here it is:

```sdl
abstract type HasCoffins {
  required coffins: int16 {
    default := 0;
  }
}
```

Most places will not have a special vampire coffin, so the default is 0. The `coffins` property is just an `int16`, and vampires can remain close to a place if the number is 1 or greater. In the mechanics of our game we would probably give vampires an activity radius of about 100 km from a place with a coffin. That's because of the typical vampire schedule which is usually as follows:

- Wake up refreshed in the coffin after the sun goes down, get ready to leave by 8 pm to find people to terrorize.
- Feel a sense of freedom because the night has just begun, and start moving away from the safety of the coffins to find victims. A vampire might use a horse-driven carriage at 25 kph, which gives a pretty wide radius.
- Around 1 or 2 am, the vampire start to feel nervous. The sun will be up in about 5 hours. Is there enough time to get home?

So the part between 8 pm and 1 am is when the vampire is free to move away, and at 25 kph we get an activity radius of about 100 km around a coffin. At that distance, even the bravest vampire will start running back towards home by 2 am.

If we were building a more complex game, vampire terrorism on humans would be worse in the winter, when the activity radius might increase to about 150km due to longer nights... but we won't get that detailed here.

With our abstract type done, we will want to have a lot of types `extending` this. First we can have `Place` extend it, and that gives it to all the other location types such as `City` and `OtherPlace`:

```sdl
abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

Ships are also big enough to have coffins (the Demeter had 50 of them, after all) so we'll extend for `Ship` as well:

```sdl
type Ship extending HasCoffins {
  name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

If we want, we can now make a quick function to test whether a vampire can enter a place:

```sdl
function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := assert_single((select Person filter .name = person_name)),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.' 
        if has_coffins else vampire.name ++ ' cannot enter.'
    );
```

The function returns an `optional str` because it may return an empty set. You'll also notice that `person_name` in this function actually just takes a str that it uses to select a `Person`. So technically it could say something like 'Jonathan Harker cannot enter'. 

If we can't trust the user of the function to always enter a `Vampire` or `MinorVampire` object, there are some options:

- Overload the function to have two signatures, one for each type of Vampire:

```sdl
function can_enter(vampire: Vampire, place: HasCoffins) -> optional str
function can_enter(vampire: MinorVampire, place: HasCoffins) -> optional str
```

- Create an abstract type (like `type AnyVampire`) and extend it for `Vampire` and `MinorVampire`. Then `can_enter` can have this signature: 

```sdl
function can_enter(vampire: AnyVampire, place: HasCoffins) -> optional str
```

Let's learn a bit more about the `optional` keyword. Without it, you need to trust the users that the input argument will be there, because a function won't be called if the input is empty. We can illustrate this point with this simple function:

```sdl
function try(place: City) -> str
  using (
    select 'Called!'
  );
```

If we call it with this:

```edgeql
select try((select City filter .name = 'London'));`
```

Then the output is `Called!` as we expected. The function requires a City as an argument, and then ignores it and returns 'Called!' instead. 

So far so good, but the input is not optional so what happens if we type this instead?

```edgeql
select try((select City filter .name = 'Beijing'));
```

Now the output will be {} because we've never inserted any data for the city 'Beijing' in our database (nobody in Bram Stoker's Dracula ever goes to Beijing). So what if we want the function to be called in any case? We can put the keyword `optional` in front of the parameter like this:

```sdl
function try(place: optional City) -> str
  using (
    select 'Called!'
  );
```

In this case we are still ignoring the argument `place` (the `City` type) but making it optional lets the function select 'Called!' regardless of whether it finds an argument or not. Having a `City` type and not having a `City` type are both acceptable in this case, and the function gets called in either case.

{ref}`The documentation <docs:ref_sdl_function_typequal>` explains it like this: 

```
...the function is called normally when the corresponding argument is empty [...]
A notable example of a function that gets called on empty input is the coalescing operator.
```

Interesting! You'll remember the coalescing operator `??` that we first saw in Chapter 11. And when we look at {eql:op}`its signature <docs:coalesce>`, you can see the `optional` in there:

`optional anytype ?? set of anytype -> set of anytype`

So those are some ideas for how to set up your functions depending on how you think people might use them.

We're coming up to an insert, so let's migrate the schema. Here are all the schema changes from the discussion so for in this chapter:

```sdl
abstract type HasCoffins {
  required coffins: int16 {
    default := 0;
  }
}

abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}

type Ship extending HasCoffins {
  name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}

function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := assert_single((select Person filter .name = person_name)),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.' if has_coffins else vampire.name ++ ' cannot enter.'
    );
```

Now let's give London some coffins. According to the book, our heroes destroyed 29 coffins at Carfax that night, which leaves 21 in London.

```edgeql
update City filter .name = 'London'
set {
  coffins := 21
};
```

Now we can finally call up our function and see if it works:

```edgeql
select can_enter('Count Dracula', (select City filter .name = 'London'));
```

We get `{'Count Dracula can enter.'}`.

Some other possible ideas for improvement later on for `can_enter()` are:

- Move the property `name` from `Place` and `Ship` over to `HasCoffins`. Then the user could just enter a string. The function would then use it to `select` the type and then display its name, giving a result like "Count Dracula can enter London."
- Require a date in the function so that we can check if the vampire is dead or not first. For example, if we entered a date after Lucy died, it would just display something like the following:

```
vampire.name ++ ' is already dead on ' ++ <str>.date
++ ' and cannot enter ' ++ city.name`
```

## More constraints

Let's look at some more constraints. We've seen `exclusive` and `max_value` already, but there are {ref}`some others <docs:ref_std_constraints>` that we can use as well.

There is one called `max_len_value` that makes sure that a string doesn't go over a certain length. That could be good for our `PC` type. `NPC`s won't need this constraint because their names are already decided by us the creators of the game, but `max_len_value()` is good for `PC`s to make sure that players don't choose names that are too long to display. This constraint doesn't exist on the original `Person` type, so we'll also need to add the `overloaded` keyword here. The `PC` type with the new constraint now looks like this:

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement()
  }
  required number: PCNumber {
    default := sequence_next(introspect PCNumber);
  }
  overloaded required name: str {
    constraint max_len_value(30);
  }
}
```

And now `PC` objects like this one with names that are too long won't be able to be inserted anymore:

```edgeql
insert PC {
 class := Class.Rogue,
 name := "Oh man, let me tell you about this PC and his name. It all began one day when.."
};
```

The error is pretty nice and tells us everything we need to know:

```
edgedb error: ConstraintViolationError: name must be no longer 
than 30 characters.
  Detail: `value of property 'name' of object type 'default`::`PC'` 
  must be no longer than 30 characters.
```

Another convenient constraint is called `one_of`, and is sort of like a quick enum. One place in our schema where we could use it is `title: str;` in our `Person` type. You'll remember that we added that in case we wanted to generate names from various parts (first name, last name, title, degree...). This constraint could work to make sure that people don't just make up their own titles:

```sdl
title: str {
  constraint one_of('Mr.', 'Mrs.', 'Ms.', 'Lord');
}
```

For us it's probably not worth it to add a `one_of` constraint though, as there are probably too many titles throughout the book (Count, the German _Herr_, Lady, Dr., Ph.D., etc.).

Another place you could imagine using a `one_of` is in the months, because the book only goes from May to October of the same year. If we had an object type generating a date then you could have this sort of constraint inside it:

```sdl
month: int64 {
  constraint one_of(5, 6, 7, 8, 9, 10);
}
```

But that will depend on how the game works.

Now let's learn about perhaps the most interesting constraint in EdgeDB: an expression that we create ourselves!

## `expression on`: the most flexible constraint

One particularly flexible constraint is called {eql:constraint}` ``expression on`` <docs:std::expression>`, which lets us add any expression we want. After `expression on` you add the expression (in brackets) that must return `true` to create an object. In other words: "Create this object _as long as_ (insert expression here)".

Let's say we need a type `Lord` for some reason later on, and all `Lord` types must have the word 'Lord' in their name. We can constrain the type to make sure that this is always the case. For this, we will use a function called {eql:func}`docs:std::contains` that looks like this:

```sdl
std::contains(haystack: str, needle: str) -> bool
```

This function returns `{true}` if the `haystack` (a string) contains the `needle` (usually a shorter string).

We can write the constraint with `expression on` and `contains()` like this:

```sdl
type Lord extending Person {
  constraint expression on (
    contains(__subject__.name, 'Lord')
  );
}
```

`__subject__` there refers to the object itself.

Let's do a migration here because a `Lord` type could be useful later.

Now when we try to insert a `Lord` without it, it won't work:

```edgeql
insert Lord {
  name := 'Billy'
  # Other stuff..
};
```

But if the `name` is 'Lord Billy' (or 'Lord William', or 'Lord' anything), it will work.

While we're at it, let's practice doing a `select` and `insert` at the same time so we see the output of our `insert` right away. We'll change `Billy` to `Lord Billy` and say that Lord Billy (considering his great wealth) has visited every place in our database.

```edgeql
select (
  insert Lord {
    name := 'Lord Billy',
    places_visited := (select Place),
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
      default::Country {name: 'Hungary'},
      default::Country {name: 'Romania'},
      default::Country {name: 'France'},
      default::Country {name: 'Slovakia'},
      default::Castle {name: 'Castle Dracula'},
      default::City {name: 'Whitby'},
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
      default::City {name: 'Exeter'},
      default::City {name: 'London'},
    },
  },
}
```

## Setting your own error messages

Since `expression on` is so flexible, you can use it in almost any way you can imagine. But that flexibility also means that there is no built-in way to let the user know how any `expression on` is supposed to work when a constraint is violated. After all, it's anyone's guess how a user might choose to use `expression on`. And that means that the automatically generated error message we have right now is not helping the user at all. Here's the message we got when we tried to insert a `Lord` named `Billy`:

```
edgedb error: ConstraintViolationError: invalid Lord
  Detail: invalid value of object type 'default::Lord'
```

So there's no way to tell that the problem is that `name` needs `'Lord'` inside it. Fortunately, all constraints allow you to set your own error message just by opening up a block with `{}` and specifying an `errmessage`, like this: `errmessage := "All lords need 'Lord' in their name."`

Let's do that with our `Lord` type now:

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord')) {
    errmessage := "All lords need \'Lord\' in their name";
  }
}
```

If you do a migration now and try to insert a `Lord` that is just called `Billy`, the error now tells us what to do:

```
`edgedb error: ConstraintViolationError: All lords need 'Lord' in their name
  Detail: All lords need 'Lord' in their name`
```

Much better! 

## Putting backlinks into the schema

Back in Chapter 6 we removed `master` link from `MinorVampire`, because `Vampire` already has the `multi slaves` link to the `MinorVampire` type. One reason was complexity, and the other was because deleting without a deletion policy becomes impossible because they both depend on each other. But now that we know how to use backlinks, we can put `master` back in `MinorVampire` if we want. Let's follow the thought process that often leads to choosing to use a backlink.

First, here is the `MinorVampire` type at present:

```sdl
type MinorVampire extending Person {
  former_self: Person;
}
```

To add the master link again, one way to start would be with a property called `master_name` that is just a string:

```sdl
type MinorVampire extending Person {
  former_self: Person;
  required master_name: str;
};
```

Then we can use it to filter out the corresponding `Vampire` and assign it to a computed link named `master` as follows:

```sdl
type MinorVampire extending Person {
  former_self: Person;
  required master_name: str;
  link master := (
    with master_name := .master_name
    assert_single(select Vampire filter .name = master_name));
};
```

Note: this is a single link, so we needed to add `assert_single()`. However, it looks a bit verbose, and we have to trust ourselves to input `master_name` correctly — definitely not ideal. In this case there is a simpler and more robust way to add `master`: using a backlink. The syntax is the same as the syntax we used last chapter except that we don't need to specify that the link is to a `MinorVampire`, because we are already inside the `MinorVampire` type.

```sdl
type MinorVampire extending Person {
  former_self: Person;
  single master := assert_single(.<slaves[is Vampire]);
};
```

Here we have written `single` to ensure that the backlink is not a `multi link`, which requires `assert_single()`. This might not be needed if our game allows vampires to share `MinorVampire` objects. However, in this case we are sure that there will only be one Vampire master (Count Dracula is the only master vampire in the book) so we can declare it a `single link`.

And if we still want to have a shortcut for `master_name`, we can just add `property master_name := .master.name;` in the above `{}` as follows:

```sdl
type MinorVampire extending Person {
  former_self: Person;
  single master := assert_single(.<slaves[is Vampire]);
  master_name := .master.name;
};
```

Now let's do a migration and test it out. We'll make a vampire named Kain, who has two `MinorVampire` slaves named Billy and Bob.

```edgeql
insert Vampire {
  name := 'Kain',
  slaves := {
    (insert MinorVampire {
      name := 'Billy',
    }),
    (insert MinorVampire {
      name := 'Bob',
    })
  }
};
```

Now if the `MinorVampire` type works as it should, we should be able to see Kain via the `master` link inside `MinorVampire` and we won't have to use a backlink. Let's check:

```edgeql
select MinorVampire {
  name,
  master_name,
  master: {
    name
  }
} filter .name in {'Billy', 'Bob'};
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

We won't see Kain and his slaves anymore so let's get rid of them with a quick delete query. The `on source delete delete target` deletion policy on the `Vampire` policy makes this easy: just delete Kain.

```edgeql
delete Vampire filter .name = 'Kain';
```

Of course, you don't have to depend on a deletion policy to delete objects. We could have deleted all three using this query for example:

```
delete Person filter .name in {'Kain', 'Billy', 'Bob'};
```

And the advantage to this deletion is that it will return all three deleted objects and their types (and even their properties and links if we wanted to show them) instead of just a single deleted `Vampire` object with the other two deletions happening unseen to us.

```
{
  default::MinorVampire {id: d01726cc-ff45-11ed-8310-7f94ffdd6f3b},
  default::MinorVampire {id: d01768da-ff45-11ed-8310-171ca2c022eb},
  default::Vampire {id: d0170a16-ff45-11ed-8310-7b730ce4d4f2},
}
```

Finally, let's add a backlink to the `Party` type that we created in Chapter 13. That's easy! The `PC` type links to `Party` via a link called `party` so all we have to do is turn that around.

```sdl
type Party {
  name: str;
  link members := .<party[is PC];
}
```

As you can see, once you understand how to write backlinks you start to wonder how you ever got anything done without them. They're one of the best reasons to use EdgeDB.

[Here is all our code so far up to Chapter 15.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you create a type called Horse with a `required name: str` that can only be 'Horse'?

2. How would you let the user know that it needs to be called 'Horse'?

3. How would you make sure that `name` for type `NPC` is always between 5 and 30 characters in length?

   Try it first with `expression on`.

4. How would you make a function called `display_coffins` that pulls up all the `HasCoffins` objects with more than 0 coffins?

5. How would you give the `Place` type a backlink to every `Person` type that visited it?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Could Renfield be of help?_
