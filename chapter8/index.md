---
tags: Multiple Inheritance, Polymorphism
---

# Chapter 8 - Dracula takes the boat to England

We are finally away from Castle Dracula. Here is what happens in this chapter:

> A boat leaves from the city of Varna in Bulgaria, sailing into the Black Sea. It has a **captain, first mate, second mate, cook**, and **five crew**. Inside is Dracula, but they don't know that he's there. Every night Dracula leaves his coffin, and every night one of the men disappears. They become afraid but don't know what is happening or what to do. One of them tells them he saw a strange man walking around the deck, but the others don't believe him. Finally it's the last day before the ship reaches the city of Whitby in England, but the captain is alone - all the others have disappeared. The captain knows the truth now. He ties his hands to the wheel so that the ship will go straight even if Dracula finds him. The next day the people in Whitby see a ship hit the beach, and a wolf jumps off and runs onto the shore - it's Dracula in his wolf form, but they don't know that. People find the dead captain tied to the wheel with a notebook in his hand and start to read the story.
> Meanwhile, Mina and her friend Lucy are in Whitby on vacation...

## Multiple inheritance

While Dracula arrives at Whitby, let's learn about multiple inheritance. We know that you can `extend` a type on another, and we have done this many times: `Person` on `NPC`, `Place` on `City`, etc. Multiple inheritance is doing this with more than one type at the same time. We'll try this with the ship's crew. The book doesn't give them any names, so we will give them numbers instead. Most `Person` types won't need a number, so we'll create an abstract type called `HasNumber` only for types that need a number:

```sdl
abstract type HasNumber {
  required property number -> int16;
}
```

We will also remove `required` from `name` for the `Person` type. Not every `Person` type will have a name now, and we trust ourselves enough to input a name if there is one. We will of course keep it `exclusive`.

Now we can use multiple inheritance for the `Crewman` type. It's very simple: just add a comma between every type you want to extend.

```sdl
type Crewman extending HasNumber, Person {
}
```

Now that we have this type and don't need a name, it's super easy to insert our crewmen thanks to `count()`. We just do this five times:

```edgeql
insert Crewman {
  number := count(detached Crewman) + 1
};
```

So if there are no `Crewman` types, he will get the number 1. The next will get 2, and so on. So after doing this five times, we can `select Crewman {number};` to see the result. It gives us:

```
{
  default::Crewman {number: 1},
  default::Crewman {number: 2},
  default::Crewman {number: 3},
  default::Crewman {number: 4},
  default::Crewman {number: 5},
}
```

Next is the `Sailor` type. The sailors have ranks, so first we will make an enum for that:

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
```

And then we will make a `Sailor` type that uses `Person` and this `Rank` enum:

```sdl
type Sailor extending Person {
  property rank -> Rank;
}
```

Then we will make a `Ship` type to hold them all.

```sdl
type Ship {
  required property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

Now to insert the sailors we just give them each a name and choose a rank from the enum:

```edgeql
insert Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

insert Sailor {
  name := 'Petrofsky',
  rank := 'FirstMate'
};

insert Sailor {
  name := 'The Second Mate',
  rank := 'SecondMate'
};

insert Sailor {
  name := 'The Cook',
  rank := 'Cook'
};
```

Inserting the `Ship` is easy because right now every `Sailor` and every `Crewman` type is part of this ship - we don't need to `filter` anywhere.

```edgeql
insert Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

Then we can look up the `Ship` to make sure that the whole crew is there:

```edgeql
select Ship {
  name,
  sailors: {
    name,
    rank,
  },
  crew: {
    number
  },
};
```

The result is:

```
{
  default::Ship {
    name: 'The Demeter',
    sailors: {
      default::Sailor {name: 'The Captain', rank: Captain},
      default::Sailor {name: 'Petrofsky', rank: FirstMate},
      default::Sailor {name: 'The First Mate', rank: SecondMate},
      default::Sailor {name: 'The Cook', rank: Cook},
    },
    crew: {
      default::Crewman {number: 1},
      default::Crewman {number: 2},
      default::Crewman {number: 3},
      default::Crewman {number: 4},
      default::Crewman {number: 5},
    },
  },
}
```

## The sequence type

On the subject of giving types a number, EdgeDB has a type called {eql:type}`docs:std::sequence` that you may find useful. This type is defined as an "auto-incrementing sequence of int64", so an `int64` that starts at 1 and goes up every time you use it. Let's imagine a `Townsperson` type for a moment that uses it. Here's the wrong way to do it:

```sdl
type Townsperson extending Person {
  property number -> sequence;
}
```

This won't work because each `sequence` keeps a record of the most recent number, and if every type just uses `sequence` then they would share it. So the right way to do it is to extend it to another type that you give a name to, and then that type will start from 1. So our `Townsperson` type would look like this instead:

```sdl
scalar type TownspersonNumber extending sequence;

type Townsperson extending Person {
  property number -> TownspersonNumber;
}
```

The number for a `sequence` type will continue to increase by 1 even if you delete other items. For example, if you inserted five `Townsperson` objects, they would have the numbers 1 to 5. Then if you deleted them all and then inserted one more `Townsperson`, this one would have the number 6 (not 1). So this is another possible option for our `Crewman` type. It's very convenient and there is no chance of duplication, but the number increments on its own every time you insert. Well, you _could_ create duplicate numbers using `update` and `set` (EdgeDB won't stop you there) but even then it would still keep track of the next number when you do the next insert.

## Using the 'is' keyword to query multiple types

So now we have quite a few types that extend the `Person` type, many with their own properties. The `Crewman` type has a property `number`, while the `NPC` type has a property called `age`.

But this gives us a problem if we want to query them all at the same time. They all extend `Person`, but `Person` doesn't have all of their links and properties. So this query won't work:

```edgeql
select Person {
  name,
  age,
  number,
};
```

The error is `ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`.

Luckily there is an easy fix for this: we can use `is` inside square brackets to specify the type. Here's how it works:

- `.name`: this stays the same, because `Person` has this property
- `.age`: this belongs to the `NPC` type, so change it to `[is NPC].age`
- `.number`: this belongs to the `Crewman` type, so change it to `[is Crewman].number`

Now it will work:

```edgeql
select Person {
  name,
  [is NPC].age,
  [is Crewman].number,
};
```

The output is now quite large, so here's just a part of it. You'll notice that types that don't have a property or a link will return an empty set: `{}`.

```
{
  # ... /snip
  default::Crewman {name: {}, age: {}, number: 4},
  default::Crewman {name: {}, age: {}, number: 5},
  default::PC {name: 'Emil Sinclair', age: {}, number: {}},
  default::NPC {name: 'The innkeeper', age: 30, number: {}},
  default::NPC {name: 'Mina Murray', age: {}, number: {}},
  default::NPC {name: 'Jonathan Harker', age: {}, number: {}},
}
```

This is pretty good, but if we send this output somewhere as JSON it won't show us the type for each of them. To refer to an object's own type in a query in EdgeDB you can use `__type__`. Calling just `__type__` will just give a `uuid` though, so we need to add `{name}` to indicate that we want the name of the type. All types have this `name` field that you can access if you want to show the object type in a query.

```edgeql
select <json>Person {
  __type__: {
    name      # Name of the type inside module default
  },
  name, # Person.name
  [is NPC].age,
  [is Crewman].number,
};
```

Choosing the six objects from before from the output, it now looks like this:

```
{
  # ... /snip
  {
  {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": null, \"number\": 4, \"__type__\": {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": null, \"number\": 5, \"__type__\": {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": \"Emil Sinclair\", \"number\": null, \"__type__\": {\"name\": \"default::PC\"}}",
  "{\"age\": 30, \"name\": \"The innkeeper\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
  "{\"age\": null, \"name\": \"Mina Murray\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
  "{\"age\": null, \"name\": \"Jonathan Harker\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
}
```

This is officially called a {ref}`polymorphic query <docs:ref_eql_select_polymorphic>`, and is one of the best reasons to use abstract types in your schema.

## Supertypes, subtypes, and generic types

The official name for a type that gets extended by another type is a `supertype` (meaning 'above type'). The types that extend them are their `subtypes` ('below types'). Because inheriting a type gives you all of its features, `subtype is supertype` will return `{true}`. And of course, `supertype is subtype` returns `{false}` because supertypes do not inherit the features of their subtypes.

In our schema, that means that `select PC is Person` returns `{true}`, while `select Person is PC` will return `{true}` or `{false}` depending on whether the object is a `PC`.

To make a query that will show this, just add a shape query with the computed property `Person is PC` and EdgeDB will tell you:

```edgeql
select Person {
    name,
    is_PC := Person is PC,
};
```

Now how about the simpler scalar types? We know that EdgeDB is very precise in having different types for integers, floats and so on, but what if you just want to know if a number is an integer for example? Of course this will work, but it's not very satisfying:

```edgeql
with year := 1893,
select year is int16 or year is int32 or year is int64;
```

Output: `{true}`.

But fortunately these types all {ref}`extend from abstract types too <docs:ref_std_abstract_types>`, and we can use them. These abstract types all start with `any`, and are: `anytype`, `anyscalar`, `anyenum`, `anytuple`, `anyint`, `anyfloat`, `anyreal`. The only one that might make you pause is `anyreal`: this one means any real number, so both integers and floats, plus the `decimal` type.

So with that you can change the above input to `select 1893 is anyint` and get `{true}`.

## Multi in other places

We've seen `multi link` quite a bit already, and you might be wondering if `multi` can appear in other places too. The answer is yes. A `multi property` is like any other property, except that it can have more than one value. For example, our `Castle` type has an `array<int16>` for the `doors` property:

```sdl
type Castle extending Place {
  property doors -> array<int16>;
}
```

But it could do something similar like this:

```sdl
type Castle extending Place {
  multi property doors -> int16;
}
```

With that, you would insert using `{}` instead of square brackets for an array:

```edgeql
insert Castle {
  name := 'Castle Dracula',
  doors := {6, 19, 10},
};
```

The next question of course is which is best to use: `multi property`, `array`, or an object type via a link. The answer is... it depends. But here are some good rules of thumb to help you decide which to choose.

- `multi property` vs. arrays:

  How large is the data you are working with? A `multi property` is more efficient when you have a lot of data, while arrays are slower. But if you have small sets, then arrays are faster than `multi property`.

  If you want to use indexes and constraints on individual elements, then you should use a `multi property`. We'll look at indexes in Chapter 16, but for now just know that they are a way of making lookups faster.

  If order is important, than an array may be better. It's easier to keep the original order of items in an array.

- `multi property` vs. objects (via `link`)

  Here we'll start with two areas where `multi property` is better, and then two areas where objects are better.

  First negative for objects: objects are always larger, and here's why. Remember `describe type as text`? Let's look at one of our types with that again. Here's the `Castle` type:

  ```
  {
    'type default::Castle extending default::Place {
      required single link __type__ -> schema::Type {
          readonly := true;
      };
      optional single property doors -> array<std::int16>;
      required single property id -> std::uuid {
          readonly := true;
      };
      optional single property important_places -> array<std::str>;
      optional single property modern_name -> std::str;
      required single property name -> std::str;
  };',
  }
  ```

  You'll remember seeing the `readonly := true` types, which are created for each object type you make. The `__type__` link and `id` property together always make up 32 bytes.

  The second negative for objects is similar: underneath, they are more work for the computer. EdgeDB runs on top of PostgreSQL, and a `multi link` to an object needs an extra "join" (a link table + object table), but a multi property only has one. Also, a "backlink" (you'll see those in Chapter 14) takes more work as well.

  Okay, now here are two positives for objects in comparison.

  Are there other types that need to refer to the same values? If so, then it may be better to use an object to keep things consistent. That's why we eventually made `places_visited` a `multi link`, for example.

  Using objects with an `exclusive` constraint is more efficient when there is a lot of property value duplication.

So hopefully that explanation should help. You can see that you have a lot of choice, so remembering the points above should help you make a decision. Most of the time, you'll probably have a sense for which one you want.

[Here is all our code so far up to Chapter 8.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you select all the `Place` types and their names, plus the `doors` property if it's a `Castle`?

2. How would you select `Place` types with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

3. How would you do the same but only showing the results of `City` and `Country` types?

4. How would you display all the `Person` types that don't have `lover`s, with their names and their type names?

5. What needs to be fixed in this query? Hint: two things definitely need to be fixed, while one more should probably be changed to make it more readable.

   ```edgeql
   select Place {
     __type__,
     name
     [is Castle]doors
   };
   ```

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Time to meet Dr. Seward, Arthur Holmwood, and Quincey Morris...and the strange Renfield._
