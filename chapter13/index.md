---
tags: Introspection, Type Union Operator
---

# Chapter 13 - Meet the new Lucy

> This time it was too late to save Lucy, and she is dying. Suddenly she opens her eyes - they look very strange. She looks at Arthur and says “Arthur! Oh, my love, I am so glad you have come! Kiss me!” He tries to kiss her, but Van Helsing grabs him and says "Don't you dare!" It was not Lucy, but the vampire inside that was talking. She soon dies, and Van Helsing puts a golden crucifix on her lips to stop her from moving (crucifixes have that power over vampires). Unfortunately, the nurse steals the crucifix to sell when nobody is looking. A few days later there is news about a lady who is stealing and biting children - it's Vampire Lucy. The newspaper calls it the "Bloofer Lady", because the young children try to call her the "Beautiful Lady" but can't pronounce _beautiful_ right. Now Van Helsing tells the other people the truth about vampires, but Arthur doesn't believe him and becomes angry that he would say crazy things about his wife. Van Helsing says, "Fine, you don't believe me. Let's go to the graveyard together tonight and see what happens. Maybe then you will."

Looks like Lucy, an `NPC`, has become a `MinorVampire`. How should we show this in the database? Let's look at the types again first.

Right now `MinorVampire` is nothing special, just a type that extends `Person`:

```sdl
type MinorVampire extending Person {
}
```

Fortunately, according to the book she is a new "type" of person. The old Lucy is gone, and this new Lucy is now one of the `slaves` linked to the `Vampire` named Count Dracula.

So instead of trying to change the `NPC` type, we can just give `MinorVampire` an optional link to `Person`:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
}
```

It's optional because we don't always know anything about people before they were made into vampires. For example, we don't know anything about the three vampire women before Dracula found them so we can't make an `NPC` type for them.

Another way to (informally) link them is to give the same date to `last_appearance` for an `NPC` and `first_appearance` for a `MinorVampire`. First we will update Lucy with her `last_appearance`:

```edgeql
update Person filter .name = 'Lucy Westenra'
set {
  last_appearance := cal::to_local_date(1893, 9, 20)
};
```

Then we can add Lucy to the `insert` for Dracula. (If you are following along, just `delete Vampire;` and `delete MinorVampire;` first so we can practice doing this longer `insert`.)

Note the first line where we create a variable called `lucy`. We then use that to bring in all the data to make her a `MinorVampire`, which is much more efficient than manually inserting all the information. It also includes her strength: we add 5 to that, because vampires are stronger.

Here's the insert:

```edgeql
with lucy := assert_single(
    (select Person filter .name = 'Lucy Westenra')
)
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (insert MinorVampire {
      name := 'Vampire Woman 1',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 2',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 3',
    }),
    (insert MinorVampire {
      # We need to give a new name, so as not to clash with former_self.
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

This gives us:

```
{
  default::MinorVampire {
    name: 'Lucy',
    strength: 5,
    first_appearance: <cal::local_date>'1893-09-20',
  },
}
```

Other filters such as `filter .name in Person.name and .first_appearance in Person.last_appearance;` are possible too but checking if the link `exists` is easiest. We could also switch to `cal::local_datetime` instead of `cal::local_date` to get the exact time down to the minute. But we won't need to get that precise just yet.

## The type union operator: |

Another operator related to types is `|`, which is used to combine them (similar to writing `or`). This query for example pulling up all `Person` types will return true:

```edgeql
with lucy := (select Person filter .name like 'Lucy%'),
select lucy is NPC | MinorVampire | Vampire;
```

It returns true if the `Person` type selected is of type `NPC`, `MinorVampire`, or `Vampire`. Since both Lucy the `NPC` and Lucy the `MinorVampire` match any of the three types, the return value is `{true, true}`.

One cool thing about the type union operator is that you can also add it to links in your schema. Let's say for example there are other `Vampire` objects in the game, and one `Vampire` that is extremely powerful can control another less powerful vampire. Right now though a `Vampire` can only control a `MinorVampire`:

```
type Vampire extending Person {
  multi link slaves -> MinorVampire;
}
```

So to represent this change, you could just use `|` and add another type:

```
type Vampire extending Person {
  multi link slaves -> MinorVampire | Vampire;
}
```

We only have Count Dracula in our database as the main `Vampire` type so we won't change our schema in this way, but keep this `|` operator in mind in case you need it.

## On target delete

We've decided to keep the old `NPC` type for Lucy, because that Lucy will be in the game until September 1893. Maybe later `PC` types will interact with her, for example. But this might make you wonder about deleting links. What if we had chosen to delete the old type when she became a `MinorVampire`? Or more realistically, what if all `MinorVampire` types connected to a `Vampire` should be deleted when the vampire dies? We won't do that for our game, but you can do it with `on target delete`. `on target delete` means "when the target is deleted", and it goes inside `{}` after the link declaration. For this we have {ref}`four options <docs:ref_datamodel_link_deletion>`:

- `restrict`: forbids you from deleting the target object.

So if you declared `MinorVampire` like this:

```sdl
type MinorVampire extending Person {
  link former_self -> Person {
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
  link former_self -> Person;
  link master -> Vampire {
    on target delete delete source
  }
}
```

The EdgeDB documentation notes that you should be careful with using this! Using `delete source` can result in quite a few automatic deletions, so be sure to double check which types are linking and being linked to.

```
If a link uses the `delete source` policy, then deleting a target of the link will also delete the object that links to it (the source). This behavior can be used to implement cascading deletes; be careful with this power!
```

Now let's look at some tips for making queries.

## Using the 'distinct' keyword

Using `distinct` is easy: just change `select` to `select distinct` to get only unique results. We can see that right now there are quite a few duplicates in our `Person` objects if we `select Person.strength;`. The output will vary because it comes from the `random` function, but it will look something like this:

```
{5, 4, 4, 4, 4, 4, 10, 2, 2, 2, 2, 2, 2, 2, 3, 3}
```

Change it to `select distinct Person.strength;` and the output will now be `{2, 3, 4, 5, 10}`.

`distinct` works by item and doesn't unpack, so `select distinct {[7, 8], [7, 8], [9]};` will return `{[7, 8], [9]}` and not `{7, 8, 9}`.

## Getting `__type__` all the time

We saw that we can use `__type__` to get object types in a query, and that `__type__` always has `.name` that shows us the type's name (otherwise we will only get the `uuid`). In the same way that we can get all the names with `select Person.name`, we can get all the type names like this:

```edgeql
select Person.__type__ {
  name
};
```

It shows us all the types attached to `Person` so far:

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

Or we can use it in a regular query to return the types as well. Let's see what types there are that have the name `Lucy`:

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
  default::NPC {__type__: schema::ObjectType {name: 'default::NPC'}, name: 'Lucy Westenra'},
  default::MinorVampire {__type__: schema::ObjectType {name: 'default::MinorVampire'}, name: 'Lucy'},
}
```

Using `__type__` to display type information is useful when the results don't include this information, for example, when the results are in JSON format. There is a setting you can use to switch format to JSON: just type `\set output-format json-pretty`. If you do that and repeat the previous query, you'll get:

```
{"__type__": {"name": "default::NPC"}, "name": "Lucy Westenra"}
{"__type__": {"name": "default::MinorVampire"}, "name": "Lucy"}
```

To restore the default format type: `\set output-format default`. Because it's so convenient, this book will show results as given with the default format options.

## Being introspective

The keyword `introspect` allows us to see more details about types. Every type has the following fields that we can access: `name`, `properties` and `links`, and `introspect` lets us see them. Let's give that a try and see what we get. We'll start with this on our `Ship` type, which is fairly small but has all three. Here are the properties and links of `Ship` again so we don't forget:

```sdl
type Ship {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

First, here is the simplest `introspect` query:

```edgeql
select (introspect Ship);
```

This query isn't very useful to us but it does show how it works: it returns `{schema::ObjectType {id: 28e74d09-0209-11ec-99f6-f587a1696697}}`. Note that `introspect` and the type go inside brackets; it's sort of a `select` expression for types that you then select again to capture.

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
2. The properties, and their names. But we also use `target`, which is what a property points to (the part after the `->`). For example, the target of `property name -> str` is `std::str`. And we want the target name too; without it we'll get an output like `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}`.
3. The links and their names, and the targets to the links...and the names of _their_ targets too.

With all that together, we get something readable and useful. The output looks like this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: '__type__', target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'crew', target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: 'sailors', target: schema::ObjectType {name: 'default::Sailor'}},
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

## The sequence type

On the subject of giving types a number, EdgeDB has a type called {eql:type}`docs:std::sequence` that you may find useful. This type is defined as an "auto-incrementing sequence of int64", so an `int64` that starts at 1 and goes up every time you use it.

A `sequence` is used as an abstract type for other type names to extend and can't be used on its own. So if we were to make a `Townsperson` type with a `sequence` property called `number`, this wouldn't quite work:

```sdl
type Townsperson extending Person {
  required property number -> sequence;
}
```

Instead, you can extend a `sequence` to another type that you give a name to, and then that type will start from 1. So our `Townsperson` type would look like this instead:

```sdl
scalar type TownspersonNumber extending sequence;

type Townsperson extending Person {
  required property number -> TownspersonNumber;
}
```

The number for a `sequence` type will continue to increase by 1 even if you delete other items. For example, if you inserted five `Townsperson` objects, they would have the numbers 1 to 5. Then if you deleted them all and then inserted one more `Townsperson`, this one would have the number 6 (not 1).

So this is another possible option for our `Crewman` type. It's very convenient and there is no chance of duplication, but the number increments on its own every time you insert. Well, you _could_ create duplicate numbers using `update` and `set` (EdgeDB won't stop you there) but even then it would still keep track of the next number when you do the next insert.

In our case, the crewmen on the ship do end up dying pretty quickly (unless a `PC` in the game can save the day?) but we won't be deleting them from the database so their `number` will always increment properly by using the `count()` function.

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
