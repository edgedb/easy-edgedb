---
tags: Type Annotations, Backlinks
---

# Chapter 14 - A ray of hope

> Finally there is some good news: Jonathan Harker is alive. After escaping Castle Dracula, it seems that he found his way to Buda-Pesth (Budapest) in August and then to a hospital, which sent Mina a letter. The hospital tells Mina that "He has had some fearful shock and continues to talk about wolves and poison and blood, of ghosts and demons."
>
> Mina takes a train to the hospital where Jonathan is recovering, after which they take a train back to England to the city of Exeter where they get married. Mina sends Lucy a letter from Exeter about the good news...but it arrives too late and Lucy never opens it.
>
> Meanwhile, Van Helsing continues to contact his associates in universities around Europe to search for information on vampires and their activities. The men visit the graveyard as planned and see vampire Lucy walking around. When Arthur sees her he finally believes Van Helsing, and so do the rest of the men. They now know that vampires are real, and manage to destroy her. Arthur is sad but happy to see that Lucy is no longer forced to be a vampire and can now die in peace.

Looks like we have a new city called Exeter, which is easy to add:

```edgeql
insert City {
  name := 'Exeter',
  population := 40000
};
```

That's the population of Exeter at the time (it has 130,000 people today), and it doesn't have a `modern_name` that is different from the one in the book.

We can also update the city of Buda-Pesth to add the name of the hospital where Jonathan Harker was staying. In addition, one of the universities that Van Helsing contacted for information is in the same city. Let's add them both:

```edgeql
update City filter .name = 'Buda-Pesth' 
  set { important_places := [
    'Hospital of St. Joseph and Ste. Mary',
    'Buda-Pesth University'
    ] 
  };
```

## Adding annotations to types and using @

Now that we know how to do introspection queries, we can start to give `annotations` to our types. An annotation is a string inside the type definition that gives us information about it. By default, annotations can use the titles `title` or `description`.

Let's imagine that in our game a `City` needs at least 50 buildings. Let's use `description` for this:

```sdl
type City extending Place {
  annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
  population: int64;
}
```

After migrating our schema, we can now do an `introspect` query on it. We know how to do this from the last chapter - just add `: {name}` everywhere to get the inner details. Ready!

```edgeql
select (introspect City) {
  annotations: {name}
  name,
  properties: {name},
};
```

Uh oh, not quite. The `annotations` part of the `introspect` query just says `std::description`:

```
{
  schema::ObjectType {
    annotations: {schema::Annotation {name: 'std::description'}},
    name: 'default::City',
    properties: {
      schema::Property {name: 'name'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'id'},
      schema::Property {name: 'population'},
    },
  },
}
```

Ah, of course: the `annotations: {name}` part returns the name of the _type_, which is `std::description`. In other words, it's a link, and the target of a link just tells us the kind of annotation that gets used. But we're looking for the value inside it.

This is where `@` comes in. To get the value inside we write something else: `@value`. The `@` is used to directly access the value inside (the string) instead of just the type name. Let's try one more time:

```edgeql
select (introspect City) {
  annotations: {
  name,
  @value
},
  name,
  properties: {name},
};
```

Now we see the actual annotation:

```
{
  schema::ObjectType {
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'A place with 50 or more buildings. Anything else is an OtherPlace',
      },
    },
    name: 'default::City',
    properties: {
      schema::Property {name: 'name'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'id'},
      schema::Property {name: 'population'},
    },
  },
}
```

What if we want an annotation with a different name besides `title` and `description`? That's easy, just declare with `abstract annotation` inside the schema and give it a name. We want to add a warning for other developers to read so that's what we'll call it:

```sdl
abstract annotation warning;
```

We'll imagine that it is important to use `Castle` instead of `OtherPlace` for not just castles, but castle towns too. Thanks to the new abstract annotation, now `OtherPlace` gives that information along with the other annotation:

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
}
```

Now let's migrate the schema again and do an introspect query on just its name and annotations:

```edgeql
select (introspect OtherPlace) {
  name,
  annotations: {name, @value}
};
```

And here it is:

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'A place with under 50 buildings - hamlets, small villages, etc.',
      },
      schema::Annotation {
        name: 'default::warning',
        @value: 'Castles and castle towns do not count! Use the Castle type for that',
      },
    },
  },
}
```

## Even more working with dates

A lot of characters are starting to die now, so let's think about that. We could come up with a method to see who is alive and who is dead, depending on a `cal::local_date`. First let's take a look at the `Person` objects we have so far. We can easily count them with `select count(Person)`. The `count` function will probably give you a number close to `{24}` at this point in the course.

There is also a function called {eql:func}`docs:std::enumerate` that gives tuples of the index numbers and the items in set that we give it. We'll use this to compare to our `count()` function to make sure that our number is right.

First a simple example of how to use `enumerate()`:

```edgeql
with three_things := {'first', 'second', 'third'},
select enumerate(three_things);
```

The output is:

```
{(0, 'first'), (1, 'second'), (2, 'third')}
```

Assuming we have 24 `Person` objects, let's use it with `select enumerate(Person.name);` to make sure that we have 24 results. The last index should be 23:

```
{
  (0, 'Jonathan Harker'),
  (1, 'Renfield'),
  (2, 'The innkeeper'),
  (3, 'Mina Murray'),
  (4, 'John Seward'),
  (5, 'Quincey Morris'),
  (6, 'Arthur Holmwood'),
  (7, 'Abraham Van Helsing'),
  (8, 'Lucy Westenra'),
  (9, 'Vampire Woman 1'),
  (10, 'Vampire Woman 2'),
  (11, 'Vampire Woman 3'),
  (12, 'Lucy'),
  (13, 'Count Dracula'),
  (14, 'The Captain'),
  (15, 'Petrofsky'),
  (16, 'The First Mate'),
  (17, 'The Cook'),
  (18, 'Emil Sinclair'),
}
```

There are only 19? Oh, that's right: the `Crewman` objects don't have a name so they don't show up.

The `Crewman` types are now just numbers, so let's give them each a name based on their numbers. This will be easy:

```edgeql
update Crewman
set {
  name := 'Crewman ' ++ <str>.number
};
```

So now that everyone has a name, let's use that to see if they are dead or not. The logic is simple: we input a `cal::local_date`, and if it's greater than the date for `last_appearance` then the character is dead.

```edgeql
with p := (select Person),
     date := <cal::local_date>'1893-08-16',
select (p.name, p.last_appearance, 
  'Dead on ' ++ <str>date ++ '? ' ++ <str>(date > p.last_appearance));
```

Here is the output:

```
{
  ('Lucy Westenra', <cal::local_date>'1893-09-20', 'Dead on 1893-08-16? false'),
  ('Crewman 1', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 2', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 3', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 4', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 5', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
}
```

We could of course turn this into a function if we use it enough.

## Globals

A lot of games feature time travel, and maybe our game will too. The player characters might have come from the present time, or might visit an apocalyptic future in which Dracula won and now rules from his base in London in the year 2023.

The interesting thing about time travel is that it is entirely dependent on what the player character is doing. The current year doesn't really belong in any of the objects currently in our database because they have nothing to do with what any player is doing at the moment. But we would like to reference the current year sometimes, because for example query on `City` objects should give their `name` when in the 19th century, but their `modern_name` when in the present.

So the current year doesn't belong to any type - it's a global parameter.

EdgeDB allows you to create global parameters using the `global` keyword. These must be scalars, unless the `global` is a computable in which case there are no limits. Let's give this a try by adding a `current_date` scalar to the schema. We'll also give it a `default` which can be the date when all player characters begin their quest.

```sdl
required global current_date: cal::local_date {
  default := <cal::local_date>'1893-05-13'
}
```

And once the migration is done, we can easily select it using the `global` keyword:

```edgeql
select global current_date;
```

This returns `{<cal::local_date>'1893-05-13'}`, the default value.

To change a global variable, just use the keyword `set`. Let's set `current_date` to a more modern date:

```edgeql
set global current_date := <cal::local_date>'2023-06-19';
```

The output is pretty simple:

```
OK: SET GLOBAL
```

So that was pretty easy! After all, this global is a scalar and scalars are pretty easy to work with.

Now let's run a query to get some `City` names. In this query we will make 1945 the cutoff date for old names vs. new names. Our `City` objects use the `modern_name` property from the abstract `Place` type as a hint that the name for the city has changed since the time in the book, so we will have to default to the `name` property regardless of the data if `modern_name` doesn't exist. Here is what the query looks like:

```
with use_modern := global current_date > <cal::local_date>'1945-01-01',
  select City {
  city_name := .modern_name if use_modern and exists .modern_name else .name
 };
```

Putting that together gives us our cities in their modern form: two of them are now Bistrița and Budapest.

```
{
  default::City {city_name: 'Whitby'},
  default::City {city_name: 'Munich'},
  default::City {city_name: 'Bistrița'},
  default::City {city_name: 'London'},
  default::City {city_name: 'Exeter'},
  default::City {city_name: 'Budapest'},
}
```

Now let's change `current_date` back to an older date. In addition to `set`, we can use the `reset` keyword for our use case because `current_date` has a default value.

```edgeql
reset global current_date;
```

And the output: 

```
OK: RESET GLOBAL
```

The `reset` keyword works for globals without default values too, but their default value is an empty set.

And now that the `current_date` has been reset, the same query as above for the `City` object names now shows us the names Bistritz and Buda-Pesth for two of the cities:

```
{
  default::City {city_name: 'Whitby'},
  default::City {city_name: 'Munich'},
  default::City {city_name: 'Bistritz'},
  default::City {city_name: 'London'},
  default::City {city_name: 'Exeter'},
  default::City {city_name: 'Buda-Pesth'},
}
```

## Backlinks

Finally, let's look at how to follow links in reverse direction, one of EdgeDB's most powerful and useful features. Learning to use backlinks can take a bit of effort, but it's well worth it.

We know how to get Count Dracula's `slaves` by name with something like this:

```edgeql
select Vampire {
  name,
  slaves: {
    name
  }
};
```

That shows us the following:

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Vampire Woman 1'},
      default::MinorVampire {name: 'Vampire Woman 2'},
      default::MinorVampire {name: 'Vampire Woman 3'},
      default::MinorVampire {name: 'Lucy'},
    },
  },
}
```

But what if we are doing the opposite? Namely, starting from `select MinorVampire` and wanting to access the `Vampire` type connected to it. Because right now, we can only bring up the properties that belong to the `MinorVampire` and `Person` type. Consider the following:

```edgeql
select MinorVampire {
  name,
  # master... how do we get this?
  # There's no link to Vampire inside MinorVampire...
}
```

Since there's no `master: Vampire` link, how do we go backwards to see the `Vampire` type that links to it?

This is where backlinks come in, where we use `.<` instead of `.` and specify the type we are looking for: `[is Vampire]`.

First let's move out of our `MinorVampire` query and just look at how `.<` works. Here is one example:

```edgeql
select MinorVampire.<slaves[is Vampire] {
  name,
  age
};
```

Because it goes in reverse order, it is selecting `Vampire` that has a link called `slaves` that are of type `MinorVampire`.

You can think of `MinorVampire.<slaves[is Vampire] {name, age}` as "Show the name and age of the Vampire objects with a link called `slaves` that are of type MinorVampire" - from right to left.

Here is the output:

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```

So far that's the same as just `select Vampire: {name, age}`. But it becomes very useful in our query before, where we wanted to access multiple types. Now we can select all the `MinorVampire` objects and their master:

```edgeql
select MinorVampire {
  name,
  master := .<slaves[is Vampire] {name},
};
```

And here is the human-readable version of `.<slaves[is Vampire] {name}` to help it stick in your memory: "the `Vampire` objects and their names that link back to `MinorVampire` through the link `slaves`".

Here is the output:

```
{
  default::MinorVampire {name: 'Vampire Woman 1', 
    master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Vampire Woman 2', 
    master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Vampire Woman 3', 
    master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Lucy', 
    master: {default::Vampire {name: 'Count Dracula'}}},
}
```

So why do we need `[is Vampire]` in the query anyway? We need this because there might be other objects of different types that link back to `MinorVampire` through a link called `slaves`. We can show this in a query on our `Place` type which is linked from quite a few types. What do you think this query will show?

```edgeql
select Place {
 name,
 visitors := .<places_visited
 };
```

In this case, we are asking EdgeDB to show us each and every object that is linking to each `Place` object via a link called `places_visited`. The output is quite large, so here is just one part of it:

```
{
  default::Country {
    name: 'Romania',
    visitors: {
      default::Vampire {id: 41c0bdc4-fef1-11ed-a968-cb41382f27c2},
      default::NPC {id: e03f804e-f9b4-11ed-86c0-835ec28e5d08},
    },
  },
  default::City {
    name: 'Munich',
    visitors: {
      default::PC {id: dfd4e9dc-f9b4-11ed-86c0-b32fa282657d},
      default::NPC {id: e03f804e-f9b4-11ed-86c0-835ec28e5d08},
    },
  }
}
```

You can see that Romania has been visited by a `Vampire` object (that's Dracula) and an `NPC` object (that's Jonathan Harker), while Munich has been visited by a `PC` object (Emil Sinclair) and an `NPC` object (Jonthan Harker again). So if we don't specify with `[is Vampire]` or `[is NPC]` then it will just return each and every object connected via a link called `places_visited`. This is fine, but it limits the shapes that we can make in the query. For example, there is no guarantee that any linking object will have the property `name` so this query won't work:

```
db> select Place {
.......  name,
.......  visitors := .<places_visited {name}
.......  };
error: InvalidReferenceError: object type 'std::BaseObject' has no link or property 'name'
  ┌─ <query>:3:32
  │
3 │  visitors := .<places_visited {name}
  │                                ^^^^ error
```

But if we specify the type, EdgeDB will now be able to tell if it has a certain property or not.

```edgeql
select Place {
  name,
  vampire_visitors := .<places_visited[is Vampire] {name},
  npc_visitors := .<places_visited[is NPC] {name}
};
```

And with that we get a nice output that shows backlinks from multiple concrete types. Here is part of the output:

```
{
  default::Country {
    name: 'Romania',
    vampire_visitors: {default::Vampire {name: 'Count Dracula'}},
    npc_visitors: {default::NPC {name: 'Jonathan Harker'}},
  },
  default::City {
    name: 'Munich',
    vampire_visitors: {},
    npc_visitors: {default::NPC {name: 'Jonathan Harker'}},
  },
}

One final note: this is why backlinks in EdgeDB are `multi` by default, as opposed to regular links which are `single` by default. After all, there might be a lot of objects here and there in our database that link back and it makes sense to assume that there could be a lot of them. But you can declare a backlink in your schema to be `single` if you want to insist that there can only be one object in a backlink.

[Here is all our code so far up to Chapter 14.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you display just the numbers for all the `Person` objects? e.g. if there are 20 of them, displaying `1, 2, 3..., 18, 19, 20`.

2. Using a computed backlink, how would you display 1) all the `Place` objects (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

3. Using a computed backlink, how would you display all the Person objects that will later become `MinorVampire`s?

   Hint: Remember, `MinorVampire` has a link back to the vampire's former self.

4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

5. How would you see this `note` annotation for `MinorVampire` in a query?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Time to get revenge._
