---
tags: Type Annotations, Backlinks
---

# Chapter 14 - A ray of hope

> Finally there is some good news: Jonathan Harker is alive. It seems that he managed to escape Castle Dracula, after which he found his way to Buda-Pesth (Budapest) in August and then to a hospital. The staff at the hospital are worried about Jonathan's mental health and send Mina a letter in which they say that he had "had some fearful shock and continues to talk about wolves and poison and blood, of ghosts and demons."
>
> Poor Jonathan! Sitting in the hospital he begins to wonder if he has just gone insane. Was Count Dracula real? Were the vampire women real too? Or did he just imagine the whole thing? How can Mina marry a man who is slowly going insane?
>
> Mina takes a train to the hospital where Jonathan is recovering, after which they take a train back to England to the city of Exeter where they get married. Mina sends Lucy a letter from Exeter about the good news...but it arrives too late and Lucy never opens it.
>
> Meanwhile, Van Helsing continues to contact his associates in universities around Europe to search for information on vampires and their activities. The men visit the graveyard as planned and see vampire Lucy walking around. When Arthur sees Lucy he finally believes Van Helsing, and so do the rest of the men. They now know that vampires are real, and manage to destroy her. Arthur is sad but also happy to see that Lucy is no longer forced to be a vampire and can now die in peace.

Looks like we have a new city called Exeter, which is easy to add:

```edgeql
insert City {
  name := 'Exeter',
  population := 40000
};
```

That's the population of Exeter at the time of the book (it has 130,000 people today), and it doesn't have a `modern_name` that is different from the one in the book.

We can also update the city of Buda-Pesth to add the name of the hospital where Jonathan Harker was staying. In addition, one of the universities that Van Helsing contacted for information is in the same city. Let's add them both:

```edgeql
update City filter .name = 'Buda-Pesth' 
  set { important_places := [
    'Hospital of St. Joseph and Ste. Mary',
    'Buda-Pesth University'
    ] 
  };
```

## Adding and accessing properties to links

One interesting thing about links is that they can hold properties too. Since a link doesn't exist without a connection between objects, properties on a link usually have something to do with the relationship between these objects as well. Or link properties may be a computed value that only makes sense when you have objects joined by a link.

We can add a link property to our schema too by thinking of some more vampire physics. We know that `Vampire` objects control `MinorVampire` objects, and are so powerful that `MinorVampire`s are even deleted when their controlling `Vampire` dies. We represented this in our schema by adding a deletion policy:

```edgeql
type Vampire extending Person {
  multi slaves: MinorVampire {
    on source delete delete target;
  }
}
```

Now let's add a link property as well. Both `MinorVampire`s and `Vampire`s have the property `strength`, and let's now say that a `Vampire`, with enough concentration, is able to give some of its own strength to the `MinorVampire`s that it controls. We'll call this link property `combined_strength`, and define it as the combined strength of both divided by two. So if one `Vampire` has a strength of 16 and a `MinorVampire` has a strength of 10, their combined strength will be 13 (16 plus 10, then divided by 2).

Added to the `Vampire` type, the `combined_strength` link property looks like this:

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on source delete delete target;
    property combined_strength := (Vampire.strength + .strength) / 2;
  }
}
```

And then we can add another interesting property to the `Vampire` type, called `army_strength`. This property will be the combined total of all the `combined_strength` link properties between a `Vampire` and the `MinorVampire`s it controls. So this number will be the total strength of the `Vampire`'s army when it is fully concentrating on giving as much strength as possible to its `MinorVampires`.

Here is what the `Vampire` type looks like after these changes. Notice anything in the syntax that looks different from what we've seen so far?

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on source delete delete target;
    property combined_strength := (Vampire.strength + .strength) / 2;
  }
  property army_strength := sum(.slaves@combined_strength);
}
```

That's right, there is an `@` in there! EdgeDB uses `@` to represent link properties as they are structurally somewhat different from regular properties. We don't need to get into the internal details but there are two things to keep in mind about link properties: they can only be `single`, and must always be optional.

And with those schema changes made, we can do a query on Count Dracula to see what his army looks like. Don't forget the `@` when doing queries too!

```edgeql
select Vampire {
  name,
  age,
  strength,
  slaves: {
    name,
    strength,
    @combined_strength
  },
  army_strength
};
```

The `strength` property of Dracula's vampire slaves is determined randomly so the query output will look different on your end, but it will be similar to the output below. You can see that the `MinorVampire` types that Dracula controls are significantly stronger when he concentrates to control them as directly as possible.

```
{
  default::Vampire {
    name: 'Count Dracula',
    age: 800,
    strength: 20,
    slaves: {
      default::MinorVampire {name: 'Vampire Woman 1', strength: 5, @combined_strength: {12.5}},
      default::MinorVampire {name: 'Vampire Woman 2', strength: 9, @combined_strength: {14.5}},
      default::MinorVampire {name: 'Vampire Woman 3', strength: 8, @combined_strength: {14}},
      default::MinorVampire {name: 'Lucy', strength: 9, @combined_strength: {14.5}},
    },
    army_strength: 55.5,
  },
}
```

## Adding annotations to types

Now that we know how to do introspection queries, we can start to give `annotations` to our types. An annotation is a string inside the type definition that gives us information about it when using an `introspect` query or when we put `__type__` into a query on an object type. By default, annotations can use the titles `title` or `description`.

Let's imagine that in our game a `City` needs at least 50 buildings, and we want the other developers to know this. Let's use an `annotation description` for this:

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

Let's do an `introspect` query on `City` again, but this time using the splat operator on the annotations to see what is inside.

```edgeql
select (introspect City) { annotations: {*}};
```

There it is!

```
{
  schema::ObjectType {
    annotations: {
      schema::Annotation {
        id: 6478af55-27fe-11ee-a636-8f86ec27644b,
        name: 'std::description',
        internal: false,
        builtin: true,
        computed_fields: [],
        inheritable: false,
        @is_owned: true,
        @owned: true,
        @value: 'A place with 50 or more buildings. Anything else is an OtherPlace',
      },
    },
  },
}
```

Ah, of course: the `annotations: {name}` part returns the name of the _type_, which is `std::description`. In other words, it's a link, and the target of a link just tells us the kind of annotation that gets used. But we're looking for the value inside it. And we can see from the `@` in the output above that the value of an annotation is a link property.

Let's try the query one more time:

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

And now the actual annotation shows up in the output.

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

What if we want an annotation with a different name besides `title` and `description`? This is surprisingly easy, just declare a new annotation by typing `abstract annotation` inside the schema and give it a name. We want to add a `warning` for other developers to read so that's what we'll call it:

```sdl
abstract annotation warning;
```

Maybe it is important to use `Castle` instead of `OtherPlace` for not just castles, but castle towns too. Thanks to the new abstract annotation, now `OtherPlace` gives that information along with the other annotation. Here are the two annotations to add to `OtherPlace`:

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

## Global scalars

Every game needs to be tested before it can be sold, and it's nice to have different possible modes when testing a game. Any game testers should be able to experience the game in the same way that a regular player would, but another mode with extra information would be helpful too.

Another global type could help here. We've had a global `Time` object in our database for some time now, which so far is our only global type. But globals can be scalar types too.

A global scalar isn't an object though, so changing its value is a bit different: instead, we use the `set` and `unset` keywords to work with it.

To try using a global scalar we can add an enum called `Mode`, and give it two values: `Info` or `Debug`. `Info` will be the default, while `Debug` will be the mode that provides extra information for the testers. After this we can make a global called `tester_mode`:

```sdl
scalar type Mode extending enum<Info, Debug>;

required global tester_mode: Mode {
    default := Mode.Info;
  }
```

A `required` global always needs a default value, which makes sense: a global is available across the entire database and is `required` so it must be present. The only way to ensure this is to add a default value. Fortunately, the EdgeDB compiler won't let a schema migration happen if we forget this. The error message would look like this:

```
error: required globals must have a default
  ┌─ c:\easy-edgedb\dbschema\default.esdl:9:3
  │
9 │   required global tester_mode: Mode;
  │   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ error

edgedb error: cannot proceed until .esdl files are fixed
```

With the migration done, let's make sure that the global value is there:

```edgeql
select global tester_mode;
```

The output is simple: just `{Info}`.

Changing a global scalar is easy too: just use the `set` keyword.

```edgeql
set global tester_mode := Mode.Debug;
```

The output here is simple too, just a message informing us that the value was successfully set:

```
OK: SET GLOBAL
```

The opposite of the `set` keyword is `reset`, which resets a global to its default value. In our case the default value is `Mode.Info`, but if we hadn't specified that `tester_mode` is `required` then the default value would have been `{}`, an empty set.

Pretty easy! The output below shows the sort of feedback you will see in the REPL when setting and resetting a global scalar value.

```
db> select global tester_mode;
{Info}
db> set global tester_mode := Mode.Debug;
OK: SET GLOBAL
db> select global tester_mode;
{Debug}
db> reset global tester_mode;
OK: RESET GLOBAL
```

And with this global value in place, we can now do queries that match on the `tester_mode` enum. Here is an example of a query that a tester using the `PC` named Emil Sinclair might use. During regular `Info` mode the query will only show the character's own info, but during `Debug` mode it will also show info on all the `NPC` objects as well. In a more complex schema we can imagine that this could be used to show a tester the health, location and so on of all the NPCs in a game, which could then be used to show them on a map or in a separate chart on the screen that is only visible during debug mode.

```edgeql
# Select all NPC objects in Debug mode
with info := NPC if global tester_mode = Mode.Debug 
             else <NPC>{},
  select PC {
    name,
    class,
    strength,
    locations := .places_visited.name,
    npc_info := info { 
      name, 
      strength
    }
} filter .name = 'Emil Sinclair';
```

The output is pretty short during `Info` mode, which only provides the necessary info to play the game in the same way as every other player.

```
{
  default::PC {
    name: 'Emil Sinclair',
    class: Mystic,
    strength: 2,
    locations: {},
    npc_info: {},
  },
}
```

But if you use `set global tester_mode := Mode.Debug;` then all of a sudden the same query will display all of the extra info!

```
{
  default::PC {
    name: 'Emil Sinclair',
    class: Mystic,
    strength: 2,
    locations: {},
    npc_info: {
      default::NPC {name: 'Jonathan Harker', strength: 5},
      default::NPC {name: 'Renfield', strength: 10},
      default::NPC {name: 'The innkeeper', strength: 1},
      default::NPC {name: 'Mina Murray', strength: 2},
      default::NPC {name: 'Quincey Morris', strength: 4},
      default::NPC {name: 'Arthur Holmwood', strength: 4},
      default::NPC {name: 'John Seward', strength: 3},
      default::NPC {name: 'Abraham Van Helsing', strength: 1},
      default::NPC {name: 'Lucy Westenra', strength: 0},
    },
  },
}
```

You could imagine some other combinations of global modes such as a "God Mode" that also makes the character invincible - because it's annoying to try to debug test a game when other `PC`s think you are a regular player and keep killing your character without knowing that you are just there to test the game mechanics.

## Backlinks

It's finally time to learn one of EdgeDB's most powerful and useful features: backlinks. A backlink is like a regular link, except that it works in the other direction: instead of a link from your object to other objects, it's a computed property that shows the objects that link to you!

Understanding backlinks and the syntax can take a bit of effort, but it's well worth it.

First let's start by looking at some regular links. We know how to get Count Dracula's `slaves` by name with something like this:

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

But what if we are doing the opposite? Namely, starting from `select MinorVampire` and wanting to access the `Vampire` type connected to it. Because right now the `MinorVampire` type only gives us access to properties and links that belong to the `MinorVampire` and `Person` type. Imagine that we want to have a `master` link that shows the `Vampire` object that links to a `MinorVampire`...how do we do it?

```edgeql
select MinorVampire {
  name,
  # master... how do we get this?
  # There's no link to Vampire inside MinorVampire...
}
```

Since there's no `master: Vampire` link, how do we go backwards to see the `Vampire` type that links to it?

This is where backlinks come in, by doing the following:

- Select the type that is being linked back to. Here that would be `MinorVampire`, because we want to see links to `MinorVampire` objects. If you are inside a type's schema, you don't need to write the type name.
- Create a link that uses the `.<` syntax instead of `.`
- Indicate the name of the link. We want to see `slaves` links from the `Vampire` type, so type `slaves` next.
- Specify the type that we are looking for: `[is Vampire]`. This isn't strictly necessary, but helpful. We'll see why in a moment.

So let's follow these steps to make our first backlink query. We want:

- A link to `MinorVampire` objects, so `select MinorVampire`
- The link to be a backlink, so add `.<`: `select MinorVampire.<`
- To look for links via a link called `slaves`, so add `slaves`: `select MinorVampire.<slaves`
- The link to be from a `Vampire` object, so add `Vampire`: `select MinorVampire.<slaves[is Vampire]`


This will return us one or more `Vampire` objects, on which of course we can add a shape. So the query will now look like this:

```edgeql
select MinorVampire.<slaves[is Vampire] {
  name,
  age
};
```

Because it goes in reverse order, it is selecting the `Vampire` objects that have a link called `slaves` that goes back `.<` to the type `MinorVampire`.

You can think of the human readable version of `MinorVampire.<slaves[is Vampire] {name, age}` as "Show the name and age of the Vampire objects with a link called `slaves` that are of type MinorVampire" - from right to left.

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

You can see that Romania has been visited by a `Vampire` object (that's Dracula) and an `NPC` object (that's Jonathan Harker), while Munich has been visited by a `PC` object (Emil Sinclair) and an `NPC` object (Jonthan Harker again).

So if we don't specify with `[is Vampire]` or `[is NPC]` then it will just return each and every object connected via a link called `places_visited`. This is fine, but it limits the shapes that we can make in the query. For example, there is no guarantee that any linking object will have the property `name` so this query won't work:

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
```

One final note: this is why backlinks in EdgeDB are `multi` by default, as opposed to regular links which are `single` by default. After all, there might be a lot of objects here and there in our database that link back and it makes sense to assume that there could be a lot of them. But you can declare a backlink in your schema to be `single` if you want to insist that there can only be one object in a backlink.

[Here is all our code so far up to Chapter 14.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you create a global str that tells you whether vampires are currently asleep or awake?

2. Using a computed backlink, how would you display 1) all the `Place` objects (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

3. Using a computed backlink, how would you display all the Person objects that will later become `MinorVampire`s?

   Hint: Remember, `MinorVampire` has a link back to the vampire's former self.

4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

5. How would you see this `note` annotation for `MinorVampire` in a query?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Time to get revenge._
