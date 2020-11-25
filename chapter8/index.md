# Chapter 8 - Dracula takes the boat to England

We are finally away from Castle Dracula. Here is what happens in this chapter:

> A boat leaves from the city of Varna in Bulgaria, sailing into the Black Sea. It has a **captain, first mate, second mate, cook**, and **five crew**. Inside is Dracula, but they don't know that he's there. Every night Dracula leaves his coffin, and every night one of the men disappears. They become afraid but don't know what is happening or what to do. One of them tells them he saw a strange man walking around the deck, but the others don't believe him. Finally it's the last day before the ship reaches the city of Whitby in England, but the captain is alone - all the others have disappeared. The captain knows the truth now. He ties his hands to the wheel so that the ship will go straight even if Dracula finds him. The next day the people in Whitby see a ship hit the beach, and a wolf jumps off and runs onto the shore - it's Dracula in his wolf form, but they don't know that. People find the dead captain tied to the wheel with a notebook in his hand and start to read the story.

> Meanwhile, Mina and her friend Lucy are in Whitby on vacation...

## Multiple inheritance

While Dracula arrives at Whitby, let's learn about multiple inheritance. We know that you can `extend` a type on another, and we have done this many times: `Person` on `NPC`, `Place` on `City`, etc. Multiple inheritance is doing this with more than one type at the same time. We'll try this with the ship's crew. The book doesn't give them any names, so we will give them numbers instead. Most `Person` types won't need a number, so we'll create an abstract type called `HasNumber` only for types that need a number:

```
abstract type HasNumber {
    required property number -> int16;
}
```

We will also remove `required` from `name` for the `Person` type. Not every `Person` type will have a name now, and we trust ourselves enough to input a name if there is one. We will of course keep it `exclusive`.

Now we can use multiple inheritance for the `Crewman` type. It's very simple: just add a comma between every type you want to extend.

```
type Crewman extending HasNumber, Person {
}
```

Now that we have this type and don't need a name, it's super easy to insert our crewmen thanks to `count()`. We just do this five times:

```
INSERT Crewman {
  number := count(DETACHED Crewman) + 1
};
```

So if there are no `Crewman` types, he will get the number 1. The next will get 2, and so on. So after doing this five times, we can `SELECT Crewman {number};` to see the result. It gives us:

```
{
  Object {number: 1},
  Object {number: 2},
  Object {number: 3},
  Object {number: 4},
  Object {number: 5},
}
```

Next is the `Sailor` type. The sailors have ranks, so first we will make an enum for that:

```
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
```

And then we will make a `Sailor` type that uses `Person` and this `Rank` enum:

```
type Sailor extending Person {
    property rank -> Rank;
}
```

Then we will make a `Ship` type to hold them all.

```
type Ship {
  required property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

Now to insert the sailors we just give them each a name and choose a rank from the enum:

```
INSERT Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

INSERT Sailor {
  name := 'Petrofsky',
  rank := 'FirstMate'
};

INSERT Sailor {
  name := 'The First Mate',
  rank := 'SecondMate'
};

INSERT Sailor {
  name := 'The Cook',
  rank := 'Cook'
};
```

Inserting the `Ship` is easy because right now every `Sailor` and every `Crewman` type is part of this ship - we don't need to `FILTER` anywhere.

```
INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

Then we can look up the `Ship` to make sure that the whole crew is there:

```
SELECT Ship {
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
  Object {
    name: 'The Demeter',
    sailors: {
      Object {name: 'Petrofsky', rank: FirstMate},
      Object {name: 'The Second Mate', rank: SecondMate},
      Object {name: 'The Cook', rank: Cook},
      Object {name: 'The Captain', rank: Captain},
    },
    crew: {
      Object {number: 1},
      Object {number: 2},
      Object {number: 3},
      Object {number: 4},
      Object {number: 5},
    },
  },
}
```

## Using IS to query multiple types

So now we have quite a few types that extend the `Person` type, many with their own properties. The `Crewman` type has a property `number`, while the `NPC` type has a property called `age`. 

But this gives us a problem if we want to query them all at the same time. They all extend `Person`, but `Person` doesn't have all of their links and properties. So this query won't work:

```
SELECT Person {
  name,
  age,
  number,
};
```

The error is `ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`. 

Luckily there is an easy fix for this: we can use `IS` inside square brackets to specify the type. Here's how it works:

- `.name`: this stays the same, because `Person` has this property
- `.age`: this belongs to the `NPC` type, so change it to `[IS NPC].age`
- `.number`: this belongs to the `Crewman` type, so change it to `[IS Crewman].number`

Now it will work:

```
SELECT Person {
  name,
  [IS NPC].age,
  [IS Crewman].number,
};
```

The output is now quite large, so here's just a part of it. You'll notice that types that don't have a property or a link will return an empty set: `{}`.

```
{
  Object {name: 'Woman 1', age: {}, number: {}},
  Object {name: 'The innkeeper', age: 30, number: {}},
  Object {name: 'Mina Murray', age: {}, number: {}},
  Object {name: {}, age: {}, number: 1},
  Object {name: {}, age: {}, number: 2},
   # /snip
}
```

This is pretty good, but the output doesn't show us the type for each of them. To refer to self in a query in EdgeDB you can use `__type__`. Calling just `__type__` will just give a `uuid` though, so we need to add `{name}` to indicate that we want the name of the type. All types have this `name` field that you can access if you want to show the object type in a query.

```
SELECT Person {
  __type__: { 
    name      # Name of the type inside module default
  },
  name, # Person.name
  [IS NPC].age,
  [IS Crewman].number,
};
```

Choosing the five objects from before from the output, it now looks like this:

```
{
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Woman 1', age: {}, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'The innkeeper', age: 30, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'Mina Murray', age: {}, number: {}},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 1},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 2},
}
```

[Here is all our code so far up to Chapter 8.](code.md)

## Time to practice

1. How would you select all the `Place` types and their names, plus the `door` property if it's a `Castle`?

2. How would you select `Place` types with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

3. How would you do the same but only showing the results of `City` and `Country` types?

4. How would you display all the `Person` types that don't have `lover`s, with their names and their type names?

5. What needs to be fixed in this query? Hint: two things definitely need to be fixed, while one more should probably be changed to make it more readable.

```
SELECT Place {
  __type__,
  name
  [IS Castle]doors
};
```

[See the answers here.](answers.md)

Up next in Chapter 9: [Time to meet Dr. Seward, Arthur Holmwood, Quincey Morris, and the strange Renfield.](../chapter9/index.md)
