---
tags: Introspection, Type Union Operator
---

# Chapter 13 - Meet the new Lucy

> This time it was too late to save Lucy, and she is dying. Suddenly she opens her eyes, but they look very strange. She looks at Arthur and says “Arthur! Oh, my love, I am so glad you have come! Kiss me!”
>
> Arthur tries to kiss her, but Van Helsing grabs him and says "Don't you dare!" It was not Lucy, but the vampire spirit inside her that was talking. Lucy soon dies, and Van Helsing puts a golden crucifix on her lips to stop her from moving (crucifixes have that power over vampires). Unfortunately, the nurse steals the crucifix to sell when nobody is looking and vampire Lucy is able to move again...
>
> A few days later there is news about a lady who is stealing and biting children. The newspaper calls it the "Bloofer Lady", because the young children try to call her the "Beautiful Lady" but can't pronounce _beautiful_ right.
>
> Van Helsing decides that it's time to tell the other people the truth about vampires, but Arthur doesn't believe him and becomes angry that he would say crazy things about his wife. Van Helsing says, "Fine, you don't believe me. Let's go to the graveyard together tonight and see what happens. Maybe then you will."

Looks like Lucy, an `NPC`, has become a `MinorVampire`. How should we show this in the database? Let's look at the types again first.

Right now `MinorVampire` is nothing special, just a type that extends `Person`:

```sdl
type MinorVampire extending Person;
```

Fortunately for us, according to the book she is a new "type" of person. The old Lucy is gone, and this new Lucy is now one of the `slaves` linked to the `Vampire` object named Count Dracula. We can treat them as two separate objects.

So instead of trying to change the `NPC` type, we can just give `MinorVampire` an optional link to `Person`:

```sdl
type MinorVampire extending Person {
  former_self: Person;
}
```

The `former_self` link isn't `required` because we don't always know anything about people before they were made into vampires. For example, we don't know anything about the three vampire women before Dracula found them so we can't make an `NPC` type for them.

Another way to (informally) link them is to give the same date to `last_appearance` for an `NPC` and `first_appearance` for a `MinorVampire`. First we will update Lucy with her `last_appearance`:

```edgeql
update Person filter .name = 'Lucy Westenra'
set {
  last_appearance := cal::to_local_date(1893, 9, 20)
};
```

After doing the migration, let's practice a big insert to add Dracula along with all of the `MinorVampire` objects. We haven't done much our existing objects so let's just  `delete Vampire;` and `delete MinorVampire;` and insert them all again.

Note the first line of the insert where we create a variable called `lucy`. We can then use that to bring in all the data to make her a `MinorVampire`, which is much more efficient than manually inserting all the information. It also includes her strength: we should add 5 to that, because vampires are generally stronger than humans.

We could give her the name 'Lucy Westenra' here because the `name` property is a delegated constraint from the `Person` type, but we'll just call her Lucy now.

Here's the insert:

```edgeql
with lucy := assert_single(
    (select Person filter .name = 'Lucy Westenra')
)
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  strength := 20,
  slaves := {
    (insert MinorVampire {
      name := 'Vampire Woman 1',
      strength := <int16>round(random() * 5) + 5
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 2',
      strength := <int16>round(random() * 5) + 5
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 3',
      strength := <int16>round(random() * 5) + 5
    }),
    (insert MinorVampire {
      name := 'Lucy',
      former_self := lucy,
      first_appearance := lucy.last_appearance,
      strength := lucy.strength + 5,
    }),
  },
  places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'})
};
```

And thanks to the `former_self` link, it's easy to find all the minor vampires that come from `Person` objects. Just filter by `exists .former_self`:

```edgeql
select MinorVampire {
  name,
  strength,
  first_appearance,
} filter exists .former_self;
```

This gives us the following input (though strength will vary):

```
{
  default::MinorVampire {
    name: 'Lucy',
    strength: 9,
    first_appearance: <cal::local_date>'1893-09-20',
  },
}
```

Other filters such as `filter .name in Person.name and .first_appearance in Person.last_appearance;` are possible too but checking if the link `exists` is easiest. We could also switch to `cal::local_datetime` instead of `cal::local_date` to get the exact time down to the minute. But we won't need to get that precise just yet.

<!-- 
-->

## The type union operator: |

Another operator related to types is `|`, which is used to combine them (similar to writing `or`). This query for example pulling up all `Person` types will return `{true}`:

```edgeql
with lucy := (select Person filter .name like 'Lucy%'),
select lucy is NPC | MinorVampire | Vampire;
```

It returns true if the `Person` type selected is of type `NPC`, `MinorVampire`, or `Vampire`. Since Lucy the `NPC` and Lucy the `MinorVampire` match any of the three types, the return value is `{true, true}`.

But the type union operator is that you can also add it to links in your schema. Let's say for example there are other `Vampire` objects in the game, and one `Vampire` that is extremely powerful can control another less powerful vampire. Right now though a `Vampire` can only control a `MinorVampire`:

```
type Vampire extending Person {
  multi slaves: MinorVampire;
}
```

So to represent this change, you could just use `|` and add another type:

```
type Vampire extending Person {
  multi slaves: MinorVampire | Vampire;
}
```

We only have Count Dracula in our database as the main `Vampire` type so we won't change our schema in this way, but keep the `|` operator in mind in case you need it.

<!--
Content to change for when https://github.com/edgedb/edgedb/issues/5823 is solved:

But how about our `Ship` type? Let's look at it again.

```
type Ship {
  required name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

Both `sailors` and `crew` link to pretty similar objects, so this could be a good place to try out the type union operator by turning them into a single link called `crew: Sailor | Crewman;`. This will take two migrations though, because we don't want to lose any of the `crew` data. So we will first change the `Ship` to have `crew` hold both types:

```
type Ship {
  required name: str;
  multi sailors: Sailor;
  multi crew: Crewman | Sailor;
}
```
-->

## On target delete

We've decided to keep the old `NPC` object for Lucy, because that Lucy will be in the game as an `NPC` until September 1893. Other `PC` objects could interact with her as an `NPC` up to this time, for example. 

But what if we had chosen to delete her, what would have happened to the objects she is linked to? Or more realistically, what if all `MinorVampire` types connected to a `Vampire` should be deleted when the vampire dies? We won't do that for our game, but you can do it with `on target delete`. This `on target delete` means "when the target is deleted", or in other words "when the object that is linked to is deleted". It goes inside `{}` after the link declaration. For this we have {ref}`four options <docs:ref_datamodel_link_deletion>`:

- `restrict`: forbids you from deleting the target object.

So if you declared `MinorVampire` like this:

```sdl
type MinorVampire extending Person {
  former_self: Person {
    on target delete restrict;
  }
}
```

then you wouldn't be able to delete Lucy the `NPC` once she was connected to Lucy the `MinorVampire`.

- `delete source`: in this case, deleting Lucy the `Person` (the target of the link) will automatically delete Lucy the `MinorVampire` (the source of the link).

- `allow`: this one simply lets you delete the target (this is the default setting).

- `deferred restrict`: forbids you from deleting the target object, unless it is no longer a target object by the end of a transaction. So this option is like `restrict` but with a bit more flexibility.

So if you wanted to have all the `MinorVampire` types automatically deleted when their `Vampire` dies, you would add a link from `MinorVampire` to the `Vampire` type. Then you would add `on target delete delete source`. `Vampire` is the target of the link, and `MinorVampire` is the source that gets deleted. It would look like this in the schema:

```edgeql
type MinorVampire extending Person {
  former_self: Person;
  master: Vampire {
    on target delete delete source
  }
}
```

Be careful when using this! Using `delete source` can result in quite a few automatic deletions, so be sure to double check which types are linking and being linked to. As the EdgeDB documentation states:

```
If a link uses the `delete source` policy, then deleting a target 
of the link will also delete the object that links to it (the source).
This behavior can be used to implement cascading deletes;
be careful with this power!
```

Now let's look at some tips for making queries.

## Using the 'distinct' keyword

The `distinct` keyword is used if we want to remove duplicate values in a set, and is easy: just change `select` to `select distinct`. We can see that right now there are quite a few duplicate values in our `Person` objects if we `select Person.strength;`. The output will vary because it comes from the `random` function, but it will look something like this:

```
{1, 1, 7, 8, 9, 9, 5, 3, 0, 0, 5, 10, 2, 0, 
2, 5, 3, 4, 0, 1, 4, 20, 5, 5, 5, 5, 5}
```

Change it to `select distinct Person.strength;` and the output will now be:

`{0, 1, 2, 3, 4, 5, 7, 8, 9, 10, 20}`

Note that `distinct` works by item and doesn't unpack or aggregate, so something like a set of arrays will check to see if the entire array is distinct or not, not each of the values inside. Thus the following query:

```edgeql
select distinct {[7, 8], [7, 8], [9]};
```

Will return `{[7, 8], [9]}` and not `{7, 8, 9}`.

## Getting `__type__` all the time

We saw that we can use `__type__` to get object types in a query, and that `__type__` always has `.name` that shows us the type's name (otherwise we will only get the `uuid`). In the same way that we can get all the names with `select Person.name`, we can get all the type names that extend the `Person` type:

```edgeql
select Person.__type__ {
  name
};
```

The output shows us all the types attached to `Person` so far:

```
{
  schema::ObjectType {name: 'default::NPC'},
  schema::ObjectType {name: 'default::Crewman'},
  schema::ObjectType {name: 'default::MinorVampire'},
  schema::ObjectType {name: 'default::Vampire'},
  schema::ObjectType {name: 'default::PC'},
  schema::ObjectType {name: 'default::Sailor'},
}
```

Or we can use it in a regular query to see the type names as well. Let's see what types there are that have the name `Lucy`:

```edgeql
select Person {
  __type__: {
    name
  },
  name
} filter .name like 'Lucy%';
```

This shows us the objects that match, and of course they are `NPC` and `MinorVampire`.

```
{
  default::NPC {__type__: schema::ObjectType 
    {name: 'default::NPC'}, name: 'Lucy Westenra'},
  default::MinorVampire {__type__: schema::ObjectType 
    {name: 'default::MinorVampire'}, name: 'Lucy'},
}
```

## Being introspective

The word introspect literally means to "look inside", and is also an EdgeDB keyword. Using the keyword `introspect` allows us to see more details about types. Every type has the following fields that we can access: `name`, `properties` and `links`, and `introspect` lets us see them. Let's give that a try and see what we get. We'll start with this on our `Ship` type, which is fairly small but has all three. Here are the properties and links of `Ship` again so we don't forget:

```sdl
type Ship {
  name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

First, here is the simplest `introspect` query:

```edgeql
select (introspect Ship);
```

This query isn't very useful to us but it does show how it works: it returns the following.

```
{schema::ObjectType {id: 28e74d09-0209-11ec-99f6-f587a1696697}}
```

Note that `introspect` and the type go inside brackets; it's sort of a `select` expression for types that you then select again to capture.

Now let's put `name`, `properties` and `links` inside the introspection:

```edgeql
select (introspect Ship) {
  name,
  properties,
  links,
};
```

This gives us:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 28e76c59-0209-11ec-adaa-85e1ecb99e47},
      schema::Property {id: 28e94e33-0209-11ec-9818-fb533a2c495f},
    },
    links: {
      schema::Link {id: 28e87ca8-0209-11ec-9ba8-71ef0b23db38},
      schema::Link {id: 28e8ee51-0209-11ec-b47e-8fd9b07debd3},
      schema::Link {id: 29176353-0209-11ec-a6c5-797987ef08b5},
    },
  },
}
```

Just like using `select` on a type, we will only get an id even if the output contains items such as other properties and links. So we will have to specify what we want to see there as well.

So let's add some more to the query to get the information we want:

```edgeql
select (introspect Ship) {
  name,
  properties: {
    name,
    target: {
      name
    }
  },
  links: {
    name,
    target: {
      name
    },
  },
};
```

So what this will give is:

1. The type name for `Ship`,
2. The properties, and their names. But we also use `target`, which is what a property points to (the part after the `:`). For example, the target of `name: str` is `std::str`. And we want the target name too; without it we'll get an output like `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}`.
3. The links and their names, and the targets to the links...and the names of _their_ targets too.

With all that together, we get something readable and useful. The output looks like this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', 
        target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', 
        target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: '__type__', 
        target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'crew', 
        target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: 'sailors', 
        target: schema::ObjectType {name: 'default::Sailor'}},
    },
  },
}
```

This type of query seems complex but it is just built on top of adding things like {name} every time you get output that only a machine can understand.

Plus, if the query isn't too complex (like ours), you might find it easier to read without so many new lines and indentation. Here's the same query written that way, which looks much simpler now:

```edgeql
select (introspect Ship) {
  name,
  properties: {name, target: {name}},
  links: {name, target: {name}},
};
```

As of EdgeDB 3.0, the easiest way to see all the possible introspection of a type is to use the splat operator. And when using the `**` double splat operator, you will see pages and pages of output. If you want to know everything there is to know about our `Ship` type, just type `select (introspect Ship) {**};`!

## The sequence type

We made a few inserts of `Crewman` objects a few chapters ago, in which we gave them each a number. To do that, we used the `count()` function to count the number of `Crewman` objects, to which we added one:

```edgeql
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
```

This gave us a sequence of numbers.

Well, it just so happens that EdgeDB has a type called {eql:type}`docs:std::sequence` that has this incrementation built in. This type is defined as an "auto-incrementing sequence of int64", so an `int64` that starts at 1 and goes up every time you increment it.

A `sequence` is used as an abstract type for other type names to extend, which allows each one of these sequence types to increment independently of any others. Note the similarity to the enum syntax we are familiar with:

```sdl
scalar type SomeSequence extending sequence;
scalar type SomeEnumType extending enum<OptionOne, OptionTwo, OptionThree>;
```

And once it is defined, we can just stick it on our object types as a property.

```sdl
scalar type SomeSequenceNumber extending sequence;

type SomeType {
  # This wouldn't work
  # required number: sequence;

  # But this will
  required number: SomeSequenceNumber;
}
```

This sequence number could be useful for our `PC` objects, because players of our game might want us to delete their accounts. When that happens we will have to delete the `PC` object that the player used, so the data will be gone. That means that we can't use `count(PC)` as a sequence number. If we did that, then the 51st player would have the number 51, but if a `PC` object was then deleted then the next one would also be number 51! A sequence is just right for us here.

Let's experiment first. We'll add a `SomeSequenceNumber` to our schema with the following, and do a migration:

```sdl
scalar type SomeSequenceNumber extending sequence;
```

And now let's play around with this sequence type a bit before we make a `PCNumber` to put on our `PC` type as a property. Basic sequence behavior is pretty simple: you increment them with the `sequence_next()` function, and reset them with `sequence_reset()`. Inside these functions you specify the sequence type that we want to increment.

In other words, just typing `sequence_next()` won't work because EdgeDB doesn't know which sequence type we want to increment:

```
db> select sequence_next();
error: QueryError: function "sequence_next()" does not exist
  ┌─ <query>:1:8
  │
1 │ select sequence_next();
  │        ^^^^^^^^^^^^^^^ Did you want "std::sequence_next(seq: schema::ScalarType)"?
```

And typing `SomeSequenceNumber` won't work either because SomeSequenceNumber isn't an object type in our schema:

```
db> select sequence_next(SomeSequenceNumber);
error: InvalidReferenceError: object type or alias 'default::SomeSequenceNumber' does not exist
  ┌─ <query>:1:22
  │
1 │ select sequence_next(SomeSequenceNumber);
  │                      ^^^^^^^^ error
```

But did you notice that the function is expecting an argument of `ScalarType`? We saw this just above when we learned the `introspect` keyword. Let's try replacing `SomeSequenceNumber` with `introspect SomeSequenceNumber` which returns a `ScalarType`:

```
db> select sequence_next(introspect SomeSequenceNumber);
{1}
```

Success! Just add `introspect` and the `sequence_next()` and `sequence_reset()` functions will know which sequence type to increment.

Let's play around with this sequence number of ours for a bit. As you can see, it can be incremented or reset, but can't be reset to anything less than 1.

```
db> select sequence_next(introspect SomeSequenceNumber);
{2}
db> select sequence_next(introspect SomeSequenceNumber);
{3}
db> select sequence_next(introspect SomeSequenceNumber);
{4}
db> select sequence_next(introspect SomeSequenceNumber);
{5}
db> select sequence_reset(introspect SomeSequenceNumber, 10);
{10}
db> select sequence_reset(introspect SomeSequenceNumber, 0);
edgedb error: NumericOutOfRangeError: setval: value 0 is out of bounds for 
sequence "6f7e322d-ff25-11ed-95e6-558fd8f3e188_sequence" (1..9223372036854775807)
db> select sequence_reset(introspect SomeSequenceNumber, 1);
{1}
```

Finally, let's change the schema for our `PC` type to include this number. We could type `sequence_next(introspect PCNumber)` every time we insert a PC object, but it's much easier just to set `sequence_next(introspect PCNumber)` in the schema as the default value. Delete `SomeSequenceNumber` from the schema if you like, and then change the schema around `PC` to look like the following and do a migration:

```sdl
scalar type PCNumber extending sequence;

type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement()
  }
  required number: PCNumber {
    default := sequence_next(introspect PCNumber);
  }
}
```

And now let's do a quick query to see if it worked:

```edgeql
select PC {
  name, 
  number,
  created_at
};
```

And with that, the sequence incrementing just works. That was easy!

```
{
  default::PC {
    name: 'Emil Sinclair',
    number: 1,
    created_at: <datetime>'2023-05-28T10:40:53.598763Z',
  },
  default::PC {
    name: 'Max Demian',
    number: 2,
    created_at: <datetime>'2023-05-30T01:13:28.022340Z',
  },
}
```

[Here is all our code so far up to Chapter 13.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you insert an `NPC` named 'Mr. Swales' who has visited the `City` called 'York', the `Country` called 'England', and the `OtherPlace` called 'Whitby Abbey'? Try it in a single insert.

2. How readable is the result of this introspect query?

   ```edgeql
   select (introspect Ship) {
     name,
     properties,
     links
   };
   ```

3. What would be the shortest way to see what links from the `Vampire` type?

4. What do you think the output of `select distinct {1, 2} + {1, 2};` will be?

   Hint: don't forget the Cartesian multiplication.

5. What do you think the output of `select distinct {2, 2} + {2, 2};` will be?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _An old friend returns._
