---
tags: Constraints, Deleting
leadImage: illustration_03.jpg
---

# Chapter 3 - Jonathan goes to Castle Dracula

In this chapter we are going to start to think about time, as you can see from what Jonathan Harker is doing:

> Jonathan Harker has just arrived at Castle Dracula after a ride in the carriage through the mountains. The ride was terrible: there was snow, strange blue fires and wolves everywhere. It was night when he arrived. He meets with Count Dracula, goes inside, and they talk all night. Dracula leaves before the sun rises though, because vampires are hurt by sunlight. Days go by, and Jonathan still doesn't know that he's a vampire. But he does notice something strange: the castle seems completely empty. If Dracula is so rich, where are his servants? Who is making his meals that he finds on the table? But Jonathan finds Dracula's stories of history very interesting, and so far is enjoying his trip.

Now we are completely inside Dracula's castle, so this is a good time to create a `Vampire` type. We can extend it from `abstract type Person` because that type has `name` and `places_visited`, which are good for `Vampire` too. But vampires are different from humans because they can live forever. One possibility is adding `age` to `Person` so that all the other types can use it too. Then `Person' would look like this:

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  property age -> int16;
}
```

`int16` means a 16 bit (2 byte) integer, which has enough space for -32768 to +32767. That's enough for age, so we don't need the bigger `int32` or `int64` types which are much larger. We also don't want it to be a `required property`, because we don't care about everybody's age.

But we don't want `PC`s and `NPC`s to live up to 32767 years, so let's remove `age` from the abstract `Person` type and give it only to `Vampire` now. We will think about the other types later. We'll make `Vampire` a type that extends `Person`, and adds age:

```sdl
type Vampire extending Person {
  property age -> int16;
}
```

Now we can create Count Dracula. We know that he lives in Romania, but that isn't a city. This is a good time to change the `City` type. We'll change the name to `Place` and make it an `abstract type`, and then `City` can extend from it. We'll also add a `Country` type that does the same thing. Now they look like this:

```sdl
abstract type Place {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}

type City extending Place;

type Country extending Place;
```

We will need to change `places_visited` in the `Person` type now to be a `Place` instead of a `City`. After all, characters can visit more places than just cities:

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> Place;
}
```

Now it's easy to make a `Country`, just do an insert and give it a name. We'll quickly insert `Country` objects for Hungary and Romania:

```edgeql
insert Country {
  name := 'Hungary'
};
insert Country {
  name := 'Romania'
};
```

(By the way, you might have noticed that `important_places` is still an `array<str>` and would probably be better as a `multi link`. That's true, though in this tutorial we never end up using it and it just stays in the schema as an array. If this were a schema for a real game, it would probably either end up turned to a `multi link` or removed if we decide we don't need it.)

## Capturing a select expression

With these countries added, we are now ready to insert Dracula. First we will change `places_visited` in `Person` from `City` to `Place` so that it can include many things: the `City` types London and Bistritz, the `Country` type Hungary, and any other types we decide to add later that will extend `Place`.

We only know that Dracula has been in Romania, so his `places_visited` will be pretty easy: just select all the `Place` types and filter on `.name = 'Romania '`. When doing this, we put the `select` inside `()` brackets (parentheses). The parentheses are used to capture the result of the `select` query. In other words, the parentheses delimit (set the boundaries for) the `select` query which is then assigned to `places_visited`.

```edgeql
insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
  # .places_visited is the result of this select query.
};
```

The result is `{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`.

The `uuid` there is the reply from the server showing which object was just created and that we were successful.

Let's check if `places_visited` worked. We only have one `Vampire` object now, so let's `select` it:

```edgeql
select Vampire {
  places_visited: {
    name
  }
};
```

This gives us: `{default::Vampire {places_visited: {default::Country {name: 'Romania'}}}}`

Perfect.

## Adding constraints

Now let's think about `age` again. It was easy for the `Vampire` type, because they can live forever. But now we want to give `age` to `PC` and `NPC` too, who are humans who don't live forever (we don't want them living up to 32767 years). For this we can add a "constraint" (a limit). Instead of `age`, we'll give them a new type called `HumanAge`. Then we can write `constraint` on it and use {ref}`one of the functions <docs:ref_datamodel_constraints>` that it can take. We will use `max_value()`.

Here's the signature for `max_value()`:

`std::max_value(max: anytype)`

The `anytype` part is interesting, because that means it can work on types like strings too. With a constraint `max_value('B')` for example you couldn't use 'C', 'D', etc.

Now let's go back to our constraint for `HumanAge`, which is 120. The `HumanAge` type looks like this:

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

Remember, it's a scalar type because it can only have one value. Then we'll add it to the `NPC` type.

```sdl
type NPC extending Person {
  property age -> HumanAge;
}
```

It's our own type with its own name, but underneath it's an `int16` that can't be greater than 120. So if we write this, it won't work:

```edgeql
insert NPC {
    name := 'The innkeeper',
    age := 130
};
```

Here is the error: `ERROR: ConstraintViolationError: Maximum allowed value for HumanAge is 120.` Perfect.

Now if we change `age` to 30, we get a message showing that it worked: `{Object {id: 72884afc-f2b1-11ea-9f40-97b378dbf5f8}}`. Now no NPCs can be over 120 years old.

## Deleting objects

Deleting in EdgeDB is very easy: just use the `delete` keyword. It's similar to `select` in that you write `delete` and then the type, which will by default delete them all. And in the same way as `select`, if you `filter` then it will only delete the ones that match the filter.

This similarity to `select` might make you nervous, because if you type something like `select City` then it will select all of them. `delete` is the same: `delete City` deletes every object for the `City` type. That's why you should think carefully before deleting anything.

So let's give it a try. Remember our two `Country` objects for Hungary and Romania? Let's delete them:

```edgeql
delete Country;
```

We got an error telling us that deleting a `Country` is not possible because it is still referenced by `places_visited` of a `Person`.

```
ERROR: ConstraintViolationError: deletion of default::Country (7f3c611c-ff43-11eb-af59-dfe5a152a5cb) is prohibited by link target policy
  Detail: Object is still referenced in link places_visited of default::Person (7f5b25ac-ff43-11eb-af59-3f8e155c6686).
```

That's Count Dracula who visited Romania getting in the way. Let's delete him first then:

```edgeql
delete Vampire;
```

Just like an insert, it gives us the id numbers of the objects that are now deleted: `{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`. Now we can try deleting the `Country` objects:

```edgeql
delete Country;
```

We got confirmation that two `Country` objects have been deleted:

```
{
    default::Country {id: 7f2da5e6-ff43-11eb-af59-33db995c2682}, default::Country {id: 7f3c611c-ff43-11eb-af59-dfe5a152a5cb},
}
```

Okay, insert them again. Now let's delete with a filter:

```edgeql
delete Country filter .name ilike '%States%';
```

Nothing matches, so the output is `{}` - we deleted nothing. Let's try again:

```edgeql
delete Country filter .name ilike '%ania%';
```

We got a `{default::Country {id: 7f3c611c-ff43-11eb-af59-dfe5a152a5cb}}`, which is certainly Romania. Only Hungary is left. What if we want to see what we deleted? No problem - just put the `delete` inside brackets and `select` it. Let's delete all the `Country` objects again but this time we'll select it:

```edgeql
select (delete Country) {
  name
};
```

The output is `{default::Country {name: 'Hungary'}}`, showing us that we deleted Hungary. And now if we do `select Country` we get a `{}`, which confirms that we did delete them all.

(Fun fact: `delete` statements in EdgeDB are actually {ref}`syntactic sugar <docs:ref_eql_statements_delete>` for `delete (select ...)`. You'll be learning something called `limit` in the next chapter with `select` and as you do so, keep in mind that you can apply the same to `delete` too.)

Finally, let's insert Hungary and Romania again to finish the chapter. We'll leave them alone now.

[Here is all our code so far up to Chapter 3.](code.md)

<!-- quiz-start -->

## Time to practice

1. This query is trying to display every `NPC` along with the `name` plus every `City` type for each `NPC`, but it's giving an error. What is it missing?

   ```edgeql
   select NPC {
     name,
     cities := select City.name
   };
   ```

2. If the `City` type needed a required property called `population`, what would it look like? What type would 'population' be?
3. This query wants to display `name` twice for some reason but is giving an error. Can you think of a way to do it?

   ```edgeql
   select Person {
     name,
     name
   };
   ```

   (Hint: the problem is that the name `name` is being used twice)

4. People keep trying to make characters with negative ages. Can you think of a constraint that can stop this?

   Hint: the current constraint is `max_value(120);`

5. Can you insert a HumanAge type?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan: "This Count Dracula knows so much about history! I'm glad I came."_
