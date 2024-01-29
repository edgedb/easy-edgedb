---
tags: Multiple Inheritance, Polymorphism
---

# Chapter 8 - Dracula takes the boat to England

After seven chapters, we finally get to leave Castle Dracula. Let's see what is happening now:

> Far away from Castle Dracula, a boat leaves from the city of Varna in Bulgaria, sailing into the Black Sea. It has a **captain, first mate, second mate, cook**, and **five crew**. Dracula is also on the ship inside a coffin, but the crew don't know that he's there. Every night Dracula leaves his coffin, and every night one of the men disappears. They become afraid but don't know what is happening or what to do. One crewman tells the others that he saw a strange man walking around the deck, but the others don't believe him.
>
> The days go by and more and more sailors disappear...
>
> Finally it's the last day before the ship reaches the city of Whitby in England, but the captain is alone - all the others have disappeared. The captain knows the truth now. He ties his hands to the wheel so that the ship will go straight even if Dracula finds him.
>
> The next day the people in Whitby see a ship hit the beach, and a wolf jumps off and runs onto the shore - it's Dracula disguised as a wolf. People find the dead captain tied to the wheel with a notebook in his hand and start to read the story.
>
> Meanwhile, Mina and her friend Lucy are in Whitby on vacation...

## Multiple inheritance, defaults, and the overloaded keyword

While Dracula arrives at Whitby, let's learn about multiple inheritance. We know that you can `extend` a type on another, and we have done this many times: `Person` on `NPC`, `Place` on `City`, etc. Multiple inheritance means to do this with more than one type at the same time.

We'll try some multiple inheritance with the ship's crew. The book doesn't give them any names, so we will give them numbers instead. Most `Person` types won't need a number, so we'll create an abstract type called `HasNumber` only for types that need a number:

```sdl
abstract type HasNumber {
  required number: int16;
}
```

Now let's put the `Crewman` type together, which will use multiple inheritance. Multiple inheritance is very simple: just add a comma between every type you want to extend.

```sdl
type Crewman extending HasNumber, Person;
```

However, here we have a problem: we never learn the names of the crewmen in the book, but the property `name` on `Person` is required. We know that every `Crewman` object will have a `number`, so it would be nice to use this to give them a name like "Crewman 1", "Crewman 2", and so on. How can we make this happen?

We can make this happen by giving `name` a default for the `Crewman` type. To give a default value, just add `{}` after the parameter and then give an expression for the default value. So far that gives us a `Crewman` type that looks like the code below. However, a migration won't quite work yet. Can you guess why?

```sdl
type Crewman extending HasNumber, Person {
  name: str {
    default := 'Crewman ' ++ <str>.number;
  }
}
```

Fortunately, the error message tells us _exactly_ what to do.

```
error: property 'name' of object type 'default::Crewman' must be declared
 using the `overloaded` keyword because it is defined
 in the following ancestor(s): default::Person
  ┌─ c:\rust\easy-edgedb\dbschema\default.esdl:6:5
  │
6 │ ╭     name: str {
7 │ │       default := 'Crewman ' ++ <str>.number;
8 │ │     }
  │ ╰─────^ error
```

In other words, `Person` is where the definition for `name` is, and this definition doesn't include any information about a default value. We still want to use `name`, but in a slightly different way. That's where the word `overloaded` comes in. Adding `overloaded` will keep the other rules for `name` (that it must be a `str`, and that it must be `exclusive`) and will add our new specification that it will have a `default` value for the `Crewman` type. So just add `overloaded` and our work is done!

```sdl
type Crewman extending HasNumber, Person {
  overloaded name: str {
    default := 'Crewman ' ++ <str>.number;
  }
}
```

Next is the `Sailor` type. The sailors have ranks, so first we will make an enum for that:

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
```

And then we will make a `Sailor` type that uses `Person` and this `Rank` enum. The sailors in the book all have their own names, so we don't need to overload their `name` keyword.

```sdl
type Sailor extending Person {
  rank: Rank;
}
```

Then we will make a `Ship` type to hold them all. As we saw in this chapter, a `Ship` object can move on its own even if all of its sailors and crewmen are dead, so we won't make its sailors or crew `required`.

```sdl
type Ship {
  required name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

With all those changes done, let's do a migration.

Now that we have the `Crewman` type and it doesn't need a name, it's super easy to insert our crewmen thanks to `count()`. We just do this five times:

```edgeql
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
```

So if there are no `Crewman` types, he will get the number 1. The next will get 2, and so on. After the five inserts we can `select Crewman {name, number};` to see the result. It gives us:

```
{
  default::Crewman {name: 'Crewman 1', number: 1},
  default::Crewman {name: 'Crewman 2', number: 2},
  default::Crewman {name: 'Crewman 3', number: 3},
  default::Crewman {name: 'Crewman 4', number: 4},
  default::Crewman {name: 'Crewman 5', number: 5},
}
```

Now to insert the sailors we just give them each a name and choose a rank from the enum. Let's mix up the enums between `str` and proper enum input to remind ourselves that EdgeDB will accept either one.

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
  rank := Rank.SecondMate
};

insert Sailor {
  name := 'The Cook',
  rank := Rank.Cook
};
```

Finally we have to insert a `Ship` to hold them all. This `insert` is easy because right now every `Sailor` and every `Crewman` type is part of this ship - we don't need to `filter` anywhere.

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
    name,
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
      default::Sailor {name: 'The Second Mate', rank: SecondMate},
      default::Sailor {name: 'The Cook', rank: Cook},
    },
    crew: {
      default::Crewman {name: 'Crewman 1', number: 1},
      default::Crewman {name: 'Crewman 2', number: 2},
      default::Crewman {name: 'Crewman 3', number: 3},
      default::Crewman {name: 'Crewman 4', number: 4},
      default::Crewman {name: 'Crewman 5', number: 5},
    },
  },
}
```

## Using the 'is' keyword to query multiple types

We now have quite a few types that extend the `Person` type, many with their own properties. The `Crewman` type has a property `number`, while the `NPC` type has a property called `age`. But since the `Person` type itself doesn't have the properties `age` or `number`, we can't just make a `Person` shape that includes them:

```edgeql
select Person {
  name,
  age,
  number,
};
```

The error is:

```
error: InvalidReferenceError: object type 'default::Person'
has no link or property 'age'
```

It looks like the only property of the three that we can put in this query is `name`. That feels pretty limiting!

```edgeql
select Person {
  name
};
```

So is there a way to include `age` if the type is an `NPC` and `number` if the type is a `Crewman`? Yes, there is! We can use the `is` keyword inside square brackets to specify the type. Here's how it works in our query on `Person` objects:

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

The output is now quite large, so here's just a part of it. You'll notice that types that don't have a property or a link will return an empty set: `{}`. So the `Crewman` objects have an `age: {}` while other objects have a `number: {}`.

```
{
  # ... /snip
  default::Crewman {name: 'Crewman 4', age: {}, number: 4},
  default::Crewman {name: 'Crewman 5', age: {}, number: 5},
  default::PC {name: 'Emil Sinclair', age: {}, number: {}},
  default::NPC {name: 'The innkeeper', age: 30, number: {}},
  default::NPC {name: 'Mina Murray', age: {}, number: {}},
  default::NPC {name: 'Jonathan Harker', age: {}, number: {}},
}
```

This is officially called a {ref}`polymorphic query <docs:ref_eql_select_polymorphic>`, and is one of the best reasons to use abstract types in your schema.

## Using __type__ to get the type

Let's do a quick experiment with the same query as above, except with the `<json>` cast. What differences do you notice?

```edgeql
select <json>Person {
  name,
  [is NPC].age,
  [is Crewman].number,
};
```

Here is part of the output:

```
{"age": null, "name": "Emil Sinclair", "number": null}
{"age": null, "name": "Vampire Woman 1", "number": null}
{"age": null, "name": "The Captain", "number": null}
{"age": null, "name": null, "number": 1}
```

The type information is all gone! This makes sense because a JSON object is just a bunch of keys and values, and with a concrete query like `PC` this would be no problem. But this query includes various object types extending `Person` and it's hard to tell which type is which. Fortunately, we can put the type information by adding `__type__` which is used to refer to an object's own type:

```edgeql
select <json>Person {
  name,
  [is NPC].age,
  [is Crewman].number,
  __type__
};
```

The result is close to what we want, but not quite. Take a look at two of the results:

```
{
  "age": null,
  "name": "Emil Sinclair",
  "number": null,
  "__type__": {"id": "4a007f07-f91f-11ed-8096-7bf54ff85912"}
}
{
  "age": null,
  "name": "The Captain",
  "number": null,
  "__type__": {"id": "48b9bb2f-faaa-11ed-966c-6fc3482a7805"}
}
```

The `id` property shows us that they are two different types, but the type name isn't readable. To fix this, we can add the `name` property after `__type__` to display this instead of the id.

```edgeql
select <json>Person {
  name,
  [is NPC].age,
  [is Crewman].number,
  __type__: {
    name
  }
};
```

And now the two objects from out previous output have human-readble names.

```
{"age": null, "name": "Emil Sinclair", "number": null, "__type__": {"name": "default::PC"}}
{"age": null, "name": "The Captain", "number": null, "__type__": {"name": "default::Sailor"}}
```

So what is `__type__`, exactly? Well, it's a link that all objects have that are used to describe them. You can see this if you type `describe type PC as text;` (or with any other object in the schema). Inside the description returned you'll see this:

```
required single link __type__: schema::ObjectType {
    readonly := true;
};
```

Interesting! So it's just an object that can be queried like any other. Let's give it a try with `PC` and the splat operator to see everything inside:

```edgeql
select PC.__type__ {*};
```

This will show all of the properties for `ObjectType`, including the `name`:

```
{
  schema::ObjectType {
    id: c7c1983a-268c-11ee-8c82-c79bbe432a02,
    name: 'default::PC',
    internal: false,
    builtin: false,
    computed_fields: [],
    final: false,
    is_final: false,
    abstract: false,
    is_abstract: false,
    inherited_fields: [],
    from_alias: false,
    is_from_alias: false,
    expr: {},
    compound_type: false,
    is_compound_type: false,
  },
}
```

So that can be pretty useful.

But if you *really* want to understand the inner workings of EdgeDB, try the same query with the double splat operator:

```
select PC.__type__ {**};
```

This will return pages and pages of information. You'll see a link called `pointers` that points to just about everything: a link to `__type__`, a link to `strength`, a link to `is_single` and that it is a computable made from the expression `not exists .lover`...and so on and so on. If you want to get a good feel for how EdgeDB works on the inside, definitely grab a cup of coffee and give this query a try!

## Supertypes, subtypes, and generic types

The official name for a type that gets extended by another type is a `supertype` (meaning 'above type'). The types that extend them are their `subtypes` ('below types'). You can visualize it like this:

* abstract type Person (supertype, above)
* ↳ type PC (subtype, under)
* ↳ type NPC (subtype, under)

Because inheriting a type gives you all of its features, a query on objects with `subtype is supertype` will always return `{true}`. In our schema a `PC` object is always a `Person`, and an `NPC` object is always a `Person`.

Conversely, `supertype is subtype` will return `{true}` or `{false}` depending on the concrete type of each object returned. A `Person` object _might_ be a `PC` object, and it _might_ be an `NPC` object.

To make a query that will check this, just use a shape with a computed property that uses the expression `Person is PC` and EdgeDB will tell you:

```edgeql
select Person {
    name,
    is_PC := Person is PC,
};
```

The output will look like this:

```
{"name": "Emil Sinclair", "is_PC": true}
{"name": "Vampire Woman 1", "is_PC": false}
{"name": "Vampire Woman 2", "is_PC": false}
# ... and so on
```

Now how about the simpler scalar types? It's nice that EdgeDB is strict about type safety and has different types for integers, floats and so on, but what if you just want to know if a number is an integer or a float? We could check to see if an integer is one of any integer types, but this makes for a pretty awkward query:

```edgeql
with year := 1893,
select year is int16 or year is int32 or year is int64;
```

Output: `{true}`.

But fortunately these types all {ref}`extend from abstract types too <docs:ref_std_abstract_types>`, and we can use them. These abstract types all start with `any`, and are: `anytype`, `anyscalar`, `anyenum`, `anytuple`, `anyint`, `anyfloat`, `anyreal`. The only one with an unclear name is `anyreal`: this one means any real number, so both integers and floats, plus the `decimal` type.

So with that you can change the above input to `select 1893 is anyint;` and get `{true}`.

## Array vs. multi property vs. multi link

We've seen `multi` links quite a bit already, and you might be wondering if `multi` can appear in other places too. The answer is yes. A `multi` property is like any other property, except that it can have more than one value. For example, our `Castle` type has an `array<int16>` for the `doors` property:

```sdl
type Castle extending Place {
  doors: array<int16>;
}
```

But it could do something similar with a `multi` property:

```sdl
type Castle extending Place {
  multi doors: int16;
}
```

With a `multi` property you would then insert using `{}` instead of square brackets for an array:

```edgeql
insert Castle {
  name := 'Castle Dracula',
  doors := {6, 19, 10},
};
```

Which makes you wonder which is best to use: `multi` for a multi property, `array`, or an object type via a link. The answer is... it depends. But here are some good rules of thumb to help you decide which to choose.

- `multi` property vs. arrays:

  How large is the data you are working with? A `multi` property is more efficient when you have a lot of data, while arrays are slower. But if you have small sets, then arrays are faster than `multi` property.

  If you want to use indexes and constraints on individual elements, then you should use a `multi` property. We'll look at indexes in Chapter 16, but for now just know that they are a way of making lookups faster.

  If order is important, than an array may be better. It's easier to keep the original order of items in an array.

- `multi` property vs. linking to objects

  Here we'll start with two areas where a `multi` property is better, and then two areas where objects are better.

  First negative for objects: objects are always larger, and here's why. Remember `describe type as text`? Let's look at one of our types with that again. Here's the `Castle` type:

  ```
  {
    'type default::Castle extending default::Place {
      required single link __type__: schema::Type {
          readonly := true;
      };
      optional single property doors: array<std::int16>;
      required single property id: std::uuid {
          readonly := true;
      };
      optional single property important_places: array<std::str>;
      optional single property modern_name: std::str;
      required single property name: std::str;
  };',
  }
  ```

  You'll remember seeing the `readonly := true` properties, which are created for each object type you make. The `__type__` link and `id` property together always make up 32 bytes.

  The second negative for objects is similar: underneath, they are more work for the computer. EdgeDB runs on top of PostgreSQL, and a multi link to an object needs an extra "join" (a link table + object table), but a `multi` property only has one. Also, a "backlink" (you'll see those in Chapter 14) takes more work as well.

  Having said that, now here are two positives for objects in comparison.

  Are there other types that need to refer to the same values? If so, then it may be better to use an object to keep things consistent. That's why we eventually made `places_visited` a `multi` link, for example.

  Using objects with an `exclusive` constraint is more efficient when there is a lot of property value duplication.

So hopefully that explanation should help. You can see that you have a lot of choice, so remembering the points above should help you make a decision. Most of the time, you'll probably have a sense for which one you want.

[Here is all our code so far up to Chapter 8.](code.md)

<!--
Todo: demonstrate turning array<int64> into multi int64 once this resolved:
https://github.com/edgedb/edgedb/issues/5749
 -->

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
