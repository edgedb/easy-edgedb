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

One cool thing about the type union operator is that you can also add it to links in your schema. Let's say for example there are other `Vampire` objects in the game, and one `Vampire` that is extremely powerful can control another less powerful vampire. Right now though a `Vampire` can only control a `MinorVampire`:

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

## On target delete, on source delete

We've decided to keep the existing `NPC` object for Lucy, because that Lucy will be in the game as an `NPC` until September 1893. Other `PC` objects could interact with her as an `NPC` up to this time, for example. 

But what if we had chosen to delete her, what would have happened to the objects she is linked to? Or more realistically, what if all `MinorVampire` types connected to a `Vampire` should be deleted when the vampire dies? This is the way vampire physics works in Bram Stoker's book: vampires drain people of their blood and turn them into minor vampires, who are only alive because the vampire is still controlling them. But when the master vampire dies, the souls of the minor vampires are finally set free and they disappear too.

We can begin thinking about these vampire physics in our game by learning about how to set a deletion policy. We'll start with the keywords `on target delete`. This `on target delete` means "when the target is deleted", or in other words "when the object that is linked to is deleted". It goes inside `{}` after the link declaration. After this point there are some options {ref}`four options <docs:ref_datamodel_link_deletion>` to choose.

One deletion policy is `restrict`, and forbids you from deleting the target object. This is the default setting. In other words, anything you link to can't be deleted unless you specify otherwise in the schema. So when you declare a `type Vampire` like this:

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire;
}
```

It is as if you had written the following:

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on target delete restrict;
  }
}
```

We can test this out right now with an attempt to delete one of the vampire women:

```edgeql
delete MinorVampire filter .name = 'Vampire Woman 1';
```

Here is the error:

```
edgedb error: ConstraintViolationError: deletion of default::MinorVampire (db56215a-268c-11ee-ab5e-6322976b513c) is prohibited by link target policy
  Detail: Object is still referenced in link slaves of default::Vampire (db561336-268c-11ee-ab5e-b338ce4886f8).
```

Another deletion policy is `allow`, and simply allows you to delete the target. Inside the `Vampire` type it would look like this, which would let us delete any `MinorVampire` linked to a `Vampire` object.

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on target delete allow;
  }
}
```

A third deletion policy is called `delete source`, which deletes the source of a link when the target is deleted. Be careful with this deletion policy! You want to be absolutely certain when setting a policy that results in automatic deletions, because EdgeDB won't let you know about the automatic deletions that happen as a result of another deletion query. And if you have an automatic deletion policy that leads to another type that has its own automatic deletion policy...you'll end up with a cascade of deletions that maybe you didn't expect to happen.

Now in our case, using `on target delete delete source` would delete Count Dracula if we deleted the `MinorVampire` (the target) called `Vampire Woman 1`. So this schema is the opposite of what we want!

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on target delete delete source;
  }
}
```

Fortunately, to switch the target and source around we can just change the keyword `target` to `source`. This gives us a deletion policy of `on source delete delete target`, which looks like this:

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire {
    on source delete delete target;
  }
}
```

Again, you want to be careful when setting a deletion policy like this one. But our database is small and controlled, so let's add this `on source delete delete target` deletion policy to the `Vampire` type and do a migration.

Now let's give this deletion policy a quick test. We'll insert a `Vampire` named 'Alucard' who has bitten a man named Brian, and made him into a `MinorVampire`. We'll insert both together:

```
insert Vampire {
  name := 'Alucard',
  slaves := (insert MinorVampire {name := "Brian"})
};
```

Now let's make sure that both of them are in the database:

```edgeql
select Vampire {**} filter .name = 'Alucard';
```

There they are! Selecting Alucard leads us to Brian as well thanks to the link.

```
{
  default::Vampire {
    strength: {},
    last_appearance: {},
    first_appearance: {},
    degrees: {},
    title: {},
    name: 'Alucard',
    pen_name: 'Alucard',
    conversational_name: 'Alucard',
    age: {},
    is_single: true,
    id: 5d3a42da-286a-11ee-9442-9fa367e8a4c0,
    places_visited: {},
    lovers: {},
    slaves: {
      default::MinorVampire {
        strength: {},
        last_appearance: {},
        first_appearance: {},
        degrees: {},
        title: {},
        name: 'Brian',
        pen_name: 'Brian',
        conversational_name: 'Brian',
        age: {},
        is_single: true,
        id: 5d3a4a96-286a-11ee-9442-c31d83f69190,
      },
    },
  },
}
```

Now if Alucard is killed, Brian should turn to dust and vanish as well:

```edgeql
delete Vampire filter .name = 'Alucard';
```

And then let's do a query to see if we can find Brian anywhere:

```
select Person filter .name = 'Brian';
```

The query returns `{}`. Thanks to the `on source delete delete target` deletion policy, Brian is gone too!

## Adding 'if orphan' to a deletion policy

Deletion policies can be pretty tricky to get right so let's put together another concrete example of one in our schema and walk through it step by step.

PCs can join together as parties inside games to work together on a common goal. We could allow players of our game to create a party that can then be joined by anyone who is interested. To start, let's make a simple `Party` type that `PC` can link to.

```sdl
type Party {
  name: str;
}

type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement();
  }
  multi party: Party; # New link here
}
```

Easy enough! Now let's think about what the `multi party: Party` line means in practice. It has a default `on target delete restrict` placed on it, which means that we can't delete any `Party` that is linked to by a PC. Let's give this a try by adding a Party and two PC objects:

```edgeql
insert Party { name := "Ye olde party" };
insert PC {
  name := "Talloon",
  class := Class.Merchant,
  party := (select Party filter .name = "Ye olde party")
};
insert PC {
  name := "Alena",
  class := Class.Rogue,
  party := (select Party filter .name = "Ye olde party")
};
```

And now any attempt to `delete Party;` will give this error:

```
edgedb error: ConstraintViolationError: deletion of default::Party (86b32874-299c-11ee-8bd8-737485b849cd) is prohibited by link target policy
  Detail: Object is still referenced in link party of default::PC (9a6eaabe-299c-11ee-8bd8-c76e1f17a3f2).
```

We don't want old `Party` objects to just sit around in our database when no `PC`s are using them anymore, so let's set up a deletion policy to delete any `Party` when all `PC` objects linking to it are deleted.

There are two items to think about here. First of all, simply adding a `on source delete delete target` as below will not work. But let's give it a try and see what happens. First change the `PC` type to have this deletion policy and do a migration:

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement();
  }
  multi party: Party {
    on source delete delete target;
  }
}
```

And then let's try to delete both PC objects that link to the `Party` object:

```edgeql
delete PC filter .name in { "Alena", "Talloon" };
```

We get an error!

```
edgedb error: ConstraintViolationError: deletion of default::Party (86b32874-299c-11ee-8bd8-737485b849cd) is prohibited by link target policy
  Detail: Object is still referenced in link party of default::PC (a2bb7ce2-299c-11ee-8bd8-431f1e9112d6).
```

This is because EdgeDB is attempting to delete `Party` when we delete each `PC` object, but there is still an invisible `on target delete restrict` policy that prevents the `Party` object from being deleted. In other words, our query tries to delete the `PC` called Alena but can't because the `PC` called Talloon still links to the Party object.

So let's add an `on target delete allow` to the `PC` type to allow the linked to `Party` object to be deleted. This is closer to what we want, but not quite! But let's do a migration and see what happens in this case.

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement();
  }
  multi party: Party {
    on source delete delete target;
    on target delete allow;
  }
}
```

Okay, now let's delete Talloon.

```edgeql
delete PC filter .name = "Talloon";
```

Success! Talloon is now deleted. But hold on a second...where did the Party go?

```edgeql
select Party;
```

The query returns an empty set! The `PC` named Alena is still in the database but her `Party` has outright disappeared. This `Party` object was automatically deleted because that's what we instructed EdgeDB to do: adding `on source delete delete target` means "delete the target every time any object linking to it is deleted". That's not what we want.

The solution here is to add two new keywords: `if orphan`. Here is the difference once `if orphan` is added:

- `on source delete delete target` means "delete the target if any object linking to it is deleted"
- `on source delete delete target if orphan` means "delete the target if the last object linking to it is deleted".

That's what we want! So now let's change the PC type to add these two new words and do another migration:

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement();
  }
  multi party: Party {
    on source delete delete target if orphan;
    on target delete allow;
  }
}
```

Next we have a bit of work to insert the `Party` object again, link the existing `PC` named Alena to it, and then to insert the `PC` named Talloon again...

```edgeql
insert Party { name := "Ye olde party" };
update PC filter .name = "Alena" 
  set { 
    party := (select Party filter .name = "Ye olde party") 
  };
insert PC { 
  name := "Talloon",
  class := Class.Merchant,
  party := (select Party filter .name = "Ye olde party") 
  };
```

And with that we are ready to delete our PC objects one at a time again. We'll start with Talloon again:

```edgeql
delete PC filter .name = "Talloon";
```

Alena is still linking to the Party object, so it should still be there. Let's check:

```edgeql
select Party { name };
```

It sure is!

```
{default::Party {name: 'Ye olde party'}}
```

And with just a single (orphan) link left, deleting Alena the PC should delete the Party object as well. Let's try it:

```edgeql
delete PC filter .name = "Alena";
```

Now if we try `select Party;` we will get an empty set, just as we hoped!

## Set operators (distinct, intersect, except) and losing shape

We are already familiar with one of the set operators in EdgeDB: `union`. This is used to join two sets together. So `select MinorVampire.name union MinorVampire.name;` will return the following:

```
{
  'Vampire Woman 1',
  'Vampire Woman 2',
  'Vampire Woman 3',
  'Vampire Woman 1',
  'Vampire Woman 2',
  'Vampire Woman 3',
}
```

The `distinct` keyword is used if we want to remove duplicate values in a set, and is easy: just change `select` to `select distinct`. 

A quick example of a property that has a lot of duplicates is the `strength` property on the `Person` type. A quick `select Person.strength;` will show this. The output will vary because it comes from the `random` function, but it will look something like this:

```
{1, 1, 7, 8, 9, 9, 5, 3, 0, 0, 5, 10, 2, 0, 
2, 5, 3, 4, 0, 1, 4, 20, 5, 5, 5, 5, 5}
```

Change it to `select distinct Person.strength;` and now only the distinct values remain:

```
{0, 1, 2, 3, 4, 5, 10, 20}
```

Note that `distinct` works by item and doesn't unpack or aggregate, so something like a set of arrays will check to see if the entire array is distinct or not, not each of the values inside. Thus using `distinct` on the following query won't do anything!

```edgeql
select distinct {[7, 8], [7, 8], [9]};
```

It will simply return the original `{[7, 8], [9]}`, and not `{7, 8, 9}`.

The next set operator is called `intersect` and returns all the items in one set that match any item in the other set. Let's try this one out on the `places_visited` property on the `Person` type. First let's look at the places visited by all our `NPC` and `PC` objects:

```
db> select PC.places_visited.name;
{'Buda-Pesth', 'Bistritz', 'Munich'}
db> select NPC.places_visited.name;
{'Romania', 'Castle Dracula', 'Buda-Pesth', 'Bistritz', 'London', 'Munich'}
```

Now let's intersect them!

```edgeql
select PC.places_visited.name intersect NPC.places_visited.name;
```

This will simply return `{'Bistritz', 'Buda-Pesth', 'Munich'}`.

What if we want to use `intersect` on some object types and give them a shape? Let's try giving a shape to all these `Place` objects instead of just showing their name. Simply putting a `{*}` after an `intersect` might seem to be the right way to make this happen:

```edgeql
select PC.places_visited intersect NPC.places_visited {*};
```

And indeed the query does work, but the output is a bit unexpected. The shape is gone!

```
{
  default::City {id: da23b4be-268c-11ee-ab5e-0bda71669b0c},
  default::City {id: da317478-268c-11ee-ab5e-8b58f378092b},
  default::City {id: da41f9ec-268c-11ee-ab5e-e3cd443a19f2},
}
```

The issue here is that set operators don't preserve the original expression type, so they don't preserve the shape of an expression either. There is in effect no shape for us to work with.

Fortunately, the solution here is fairly simple: we can use `with` to capture the result of a set operator, and then *that* will have a shape that we can work with. So a small change to our query will do the job:

```edgeql
with common_locations := PC.places_visited intersect NPC.places_visited,
  select common_locations {*};
```

The result is what we hoped to see: not just the names of the places visited by both `PC` and `NPC` objects, but their properties too.

```
{
  default::City {
    id: da41f9ec-268c-11ee-ab5e-e3cd443a19f2,
    important_places: ['Golden Krone Hotel'],
    modern_name: 'Bistrița',
    name: 'Bistritz',
  },
  default::City {
    id: da23b4be-268c-11ee-ab5e-0bda71669b0c,
    important_places: {},
    modern_name: {},
    name: 'Munich',
  },
  default::City {
    id: da317478-268c-11ee-ab5e-8b58f378092b,
    important_places: {},
    modern_name: 'Budapest',
    name: 'Buda-Pesth',
  },
}
```

The last set operator to learn is called `except`, and it's the opposite of `intersect`. While `intersect` returns items that are in the first set as well as the other, `except` returns items from the first set that are *not* shared with the second set.

The `except` operator is a good opportunity to demonstrate that order can matter when using a set operator. Our `PC` objects have been to three cities: `{'Buda-Pesth', 'Bistritz', 'Munich'}`. The `NPC` objects have been to more places: `{'Romania', 'Castle Dracula', 'Buda-Pesth', 'Bistritz', 'London', 'Munich'}`. What do you think will happen in the query below that uses `intersect`? Note the order in which it is done.

```edgeql
select PC.places_visited.name except NPC.places_visited.name;
```

That's right, the query simply returns a `{}` empty set. That's because EdgeDB went through the three names returned by `PC.places_visited.name` as follows:

- 'Buda-Pesth': Is this one inside NPC.places_visited.name? Yes. So don't return it.
- 'Bistritz': Same.
- 'Munich': Same.

And then the query was done.

But if we reverse the query then it will return some names:

```edgeql
select NPC.places_visited.name except PC.places_visited.name;
```

The result is `{'Castle Dracula', 'London', 'Romania'}`, because this time EdgeDB began with the names of the places visited by the NPC objects and found three names that didn't have a matching value in the set of names returned by the `PC` objects.

In other words, you can sort of think of `except` as meaning `minus`.

## Being introspective

We saw back in Chapter 8 that we can use `__type__` to get object types in a query, and that `__type__` always has a `name` property that shows us the type's name (otherwise we will only see its `uuid`). In the same way that we can get all the names of `Person` objects with `select Person.name`, we can use `__type__.name` to get all the type names that extend the `Person` type:

```edgeql
select Person.__type__.name;
```

The output shows us all the names of types attached to `Person` so far, namely the types that extend `Person`:

```
{
  'default::NPC',
  'default::Sailor',
  'default::MinorVampire',
  'default::Crewman',
  'default::Vampire',
  'default::PC',
}
```

On top of the `name` property, the two most useful fields inside `__type__` are `properties` and `links`. (You can also just choose the field `pointers` which holds both `properties` and `links` together.) If we add these to the query the output will get quite long.

```edgeql
select Person.__type__ {
   name,
   properties: {name},
   links: {name}
};
```

The output contains the name, properties and links of each and every type that extends `Person`:

```
{
  schema::ObjectType {
    name: 'default::NPC',
    properties: {
      schema::Property {name: 'strength'},
      schema::Property {name: 'last_appearance'},
      schema::Property {name: 'first_appearance'},
      schema::Property {name: 'degrees'},
      schema::Property {name: 'title'},
      schema::Property {name: 'name'},
      schema::Property {name: 'pen_name'},
      schema::Property {name: 'conversational_name'},
      schema::Property {name: 'is_single'},
      schema::Property {name: 'id'},
      schema::Property {name: 'age'},
    },
    links: {
      schema::Link {name: '__type__'},
      schema::Link {name: 'places_visited'},
      schema::Link {name: 'lovers'},
    },
  },
  schema::ObjectType {
    name: 'default::Sailor',
    # And so on...
```

For such type-related queries, EdgeDB has a keyword called `introspect` that is specialized for looking inside types. (Indeed, the word *introspect* itself means to "look inside".) It's a little bit similar to adding `__type__` to a query, but is more focused and has certain abilities and uses that you can't get by using `__type__`.

A good rule of thumb is that:

- If you are doing a query on objects and want to add some type information on the object itself, adding `__type__` lets you do this.
- If you want to do a query exclusively on the type itself, go with `introspect`.

To do an `introspect` query, just wrap it in parentheses inside a `select`. Here is how we can use `introspect` to look inside the `Person` type:

```edgeql
select (introspect Person) {
    name,
    pointers: {name}
};
```

Both the query and output are now quite clean:

```
{
  schema::ObjectType {
    name: 'default::Person',
    pointers: {
      schema::Link {name: '__type__'},
      schema::Property {name: 'id'},
      schema::Link {name: 'lovers'},
      schema::Property {name: 'is_single'},
      schema::Link {name: 'places_visited'},
      schema::Property {name: 'age'},
      schema::Property {name: 'name'},
      schema::Property {name: 'title'},
      schema::Property {name: 'conversational_name'},
      schema::Property {name: 'degrees'},
      schema::Property {name: 'pen_name'},
      schema::Property {name: 'first_appearance'},
      schema::Property {name: 'last_appearance'},
      schema::Property {name: 'strength'},
    },
  },
}
```

Using `introspect` also lets us look inside scalar types, which isn't possible with `__type__` (which only works on object types). So this query won't work:

```edgeql
select Class.__type__ {*};
```

But `introspect` will:

```edgeql
select (introspect Class);
```

That query just returns a `{schema::ScalarType {id: c7c181cc-268c-11ee-980e-a10f818aefc0}}`. But `ScalarType` looks like an object type of its own that we can use the splat operator on! Let's see what's inside:

```edgeql
select (introspect Class) {*};
```

There it is! Lots of info about our `Class` enum:

```
{
  schema::ScalarType {
    final: false,
    is_final: false,
    abstract: false,
    is_abstract: false,
    id: c7c181cc-268c-11ee-980e-a10f818aefc0,
    name: 'default::Class',
    internal: false,
    builtin: false,
    computed_fields: [],
    expr: {},
    from_alias: false,
    is_from_alias: false,
    inherited_fields: [],
    default: {},
    enum_values: ['Rogue', 'Mystic', 'Merchant'],
    arg_values: {},
  },
}
```

You can even `introspect` the most basic of EdgeDB's scalar types:

```edgeql
select ((introspect str), (introspect int64), (introspect int16));
```

The output for this query is actually pretty interesting. We can see that the `id`s for EdgeDB's basic scalar types have been manually chosen instead of automatically generated.

```
{
  (
    schema::ScalarType {id: 00000000-0000-0000-0000-000000000101},
    schema::ScalarType {id: 00000000-0000-0000-0000-000000000105},
    schema::ScalarType {id: 00000000-0000-0000-0000-000000000103},
  ),
}
```

But the `introspect` keyword isn't limited to doing queries on types for our own information and fun. EdgeDB has one type that requires you to `introspect` it whenever it gets used! Let's take a look at that now.

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

And now let's play around with this sequence type a bit before we make a `PCNumber` to put on our `PC` type as a property. Basic sequence behavior is pretty simple: you increment them with the `sequence_next()` function, and reset them with `sequence_reset()`. But here's an important point: inside these functions you need to specify the sequence type that we want to increment.

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

But did you notice that the function is expecting an argument of `ScalarType`? We saw this just above when we learned the `introspect` keyword. So let's try replacing `SomeSequenceNumber` with `introspect SomeSequenceNumber` which returns a `ScalarType`:

```
db> select sequence_next(introspect SomeSequenceNumber);
{1}
```

Success! Just add `introspect` and the `sequence_next()` and `sequence_reset()` functions will know which sequence type to increment.

Now that we know how sequences work, let's play around with this sequence number of ours for a bit. As you can see, it can be incremented or reset, but can't be reset to anything less than 1.

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

And with that, the sequence incrementing just works. That was easy! And thanks to the default value we provided, EdgeDB has even updated our existing object types with the `number` property.

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

3. What would be the shortest way to see what links form the `Vampire` type?

4. What do you think the output of `select distinct {1, 2} + {1, 2};` will be?

   Hint: don't forget the Cartesian multiplication.

5. What do you think the output of `select distinct {2, 2} + {2, 2};` will be?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _An old friend returns._
