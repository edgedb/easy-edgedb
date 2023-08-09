---
tags: Aliases, Named Tuples, Mutation Rewrites
---

# Chapter 17 - Poor Renfield. Poor Mina.

> Last chapter Dr. Seward and Dr. Van Helsing wanted to let Renfield out, but couldn't trust him. But it turns out that Renfield was telling the truth! Dracula had found out that night that the heroes were destroying his coffins and decided to attack Mina. Dracula succeeded, and now Mina is slowly turning into a vampire. She is still human, but has a connection with Dracula now.
>
> The group finds Renfield in a pool of blood, dying. Renfield is sorry and tells them the truth. Renfield had been in communication with Dracula because he wanted to become a vampire too, so he let Dracula into the house. But once inside Dracula ignored Renfield, and headed for Mina's room. Renfield attacked Dracula to try to stop him from hurting Mina, but Dracula was much stronger and won.
>
> Mina has not given up though, and has an idea. If she is now connected to Dracula, what happens if Van Helsing uses hypnotism on her? Could that work? Could they use the connection to Dracula to their advantage? Van Helsing takes out his pocket watch and tells her: "Please concentrate on this watch. You are beginning to feel sleepy...what do you feel? Think about the man who attacked you, try to feel where he is..."

Renfield is no longer alive, so to start the chapter we need to use `update` to give him a `last_appearance`. Let's do a fancy one where we grab the output from `update` to display some interesting properties, including one where we use `last_appearance` minus `first_appearance` to show how long the character Renfield appears in the book:

```edgeql
with updated := (update NPC filter .name = 'Renfield' set { last_appearance := <cal::local_date>'1893-10-03' })
  select updated {
    name,
   first_appearance,
   last_appearance,
   days_in_book := .last_appearance - .first_appearance
};
```

This gives us:

```
{
  default::NPC {
    name: 'Renfield',
    first_appearance: <cal::local_date>'1893-05-26',
    last_appearance: <cal::local_date>'1893-10-03',
    days_in_book: <cal::date_duration>'P130D',
  },
}
```

## Building up abstract types

Wherever there are vampires, there are vampire hunters. Sometimes they will destroy a vampire's coffins, and other times vampires will build more. It would be nice to have a generic way to update this information. But the problem right now is this:

- the `HasCoffins` type is an abstract type, with one property: `coffins`
- places that can have coffins are `Place` and all the types from it, plus `Ship`,
- the best way to filter is by `.name`, but `HasCoffins` doesn't have this property.

So maybe it's time to turn this abstract type into a larger one called `HasNameAndCoffins`, and put the `name` and `coffins` properties inside there. This won't be a problem because every place needs a name and a number of coffins in our game. Remember, 0 coffins means that vampires can't stay in a place for long: just quick trips in at night before the sun rises. It's essentially a "Has name and can vampires terrorize it" property. And that will let us do queries on `HasNameAndCoffins` which is guaranteed to have these two properties.

Here is the type with its new property. We'll give it two constraints: `exclusive` and `max_len_value` to keep names from being too long.

```sdl
abstract type HasNameAndCoffins {
  required coffins: int16 {
    default := 0;
  }
  required name: str {
    delegated constraint exclusive;
    constraint max_len_value(30);
  }
}
```

So now we can change our `Ship` type (notice that we removed `name`):

```sdl
type Ship extending HasNameAndCoffins {
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

And the `Place` type. It's much simpler now.

```sdl
abstract type Place extending HasNameAndCoffins {
  modern_name: str;
  multi important_places: Landmark;
}
```

Finally, we can change our `can_enter()` function. This one needed a `HasCoffins` type before:

```sdl
function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := (select Person filter .name = person_name),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.'
        if has_coffins else vampire.name ++ ' cannot enter.'
    );
```

But now that `HasNameAndCoffins` holds `name`, the user can now just enter a string. We'll change it to this:

```sdl
function can_enter(person_name: str, place: str) -> optional str
  using (
    with
      vampire := assert_single(
        (select Person filter .name = person_name)
      ),
      enter_place := assert_single(
        (select HasNameAndCoffins filter .name = place)
      )
    select vampire.name ++ ' can enter.' if enter_place.coffins > 0 
      else vampire.name ++ ' cannot enter.'
  );
```

And now let's do a migration. The migration questions this time have some interesting ones, which are:

```
did you drop function 'default::can_enter'? [y,n,l,c,b,s,q,?]
> y
did you create function 'default::can_enter'? [y,n,l,c,b,s,q,?]
> y
did you alter object type 'default::Place'? [y,n,l,c,b,s,q,?]
> y
did you alter object type 'default::Ship'? [y,n,l,c,b,s,q,?]
> y
The following extra DDL statements will be applied:
    ALTER TYPE default::Ship {
        ALTER PROPERTY name {
            RESET OPTIONALITY;
            DROP OWNED;
            RESET TYPE;
        };
    };
```

The CLI first asks us if we dropped the function `default::can_enter`, which we might be tempted to say no to - because from our point of view we changed, not dropped, this function. But remember that from the compiler's point of view a function with a different signature is a different function, so we are effectively dropping it and creating another one. And that means that the fact that they both have the same `can_enter` name is irrelevant!

The next few questions show that EdgeDB understands that we are using an abstract type to hold the `name` property that `Place` and `Ship` have held themselves all this time. It does this with a few extra DDL commands that alter the `name` property from one owned by `Ship` to one that is still inside the `Ship` type, just not owned anymore. The migration file holds the rest of the DDL commands, some of which are:

```ddl
  ALTER TYPE default::Place {
      DROP EXTENDING default::HasCoffins;
      EXTENDING default::HasNameAndCoffins LAST;
      ALTER PROPERTY name {
          ALTER CONSTRAINT std::exclusive {
              RESET DELEGATED;
              DROP OWNED;
          };
          RESET OPTIONALITY;
          DROP OWNED;
          RESET TYPE;
      };
  };
  ALTER TYPE default::Ship {
      DROP EXTENDING default::HasCoffins;
      EXTENDING default::HasNameAndCoffins LAST;
  };
  ALTER TYPE default::Ship {
      ALTER PROPERTY name {
          RESET OPTIONALITY;
          DROP OWNED;
          RESET TYPE;
      };
  };
```

How nice that we don't have to type all of that ourselves! And if we do a query on the `Ship` and `Place` names we can see that all the data is still there.

```
db> select {Ship.name, Place.name};
{
  'The Demeter',
  'Castle Dracula',
  'Rumeli Feneri',
  'Whitby',
  'Buda-Pesth',
  'Bistritz',
  'Munich',
  'Exeter',
  'London',
  'Hungary',
  'Romania',
  'France',
  'Slovakia',
}
```

Meanwhile, with the new function in our schema we can just enter `can_enter('Count Dracula', 'Munich')` to get `'Count Dracula cannot enter.'`. That makes sense: Dracula didn't bring any coffins there.

Finally, thanks to this larger abstract type we can now put together a query that takes arguments to change the number of coffins in a number of places. It's easy:

```edgeql
update HasNameAndCoffins filter .name = <str>$place_name
set {
  coffins := .coffins + <int16>$number
};
```

Now let's give the ship `The Demeter` some coffins.

```
db> update HasNameAndCoffins filter .name = <str>$place_name
  set {
    coffins := .coffins + <int16>$number
  };
Parameter <str>$place_name: The Demeter
Parameter <int16>$number: 10
```

Castle Dracula naturally should have some coffins too. 50 feels about right.

```
db> update HasNameAndCoffins filter .name = <str>$place_name
  set {
    coffins := .coffins + <int16>$number
  };
Parameter <str>$place_name: Castle Dracula
Parameter <int16>$number: 50
```

Then we'll make sure that these places got them:

```edgeql
select HasNameAndCoffins { 
  name, 
  coffins } 
filter .coffins > 0;
```

And the result:

```
{
  default::Castle {name: 'Castle Dracula', coffins: 50},
  default::Ship {name: 'The Demeter', coffins: 20},
  default::City {name: 'London', coffins: 21},
}
```

Looks like the Demeter and Castle Dracula got their coffins!

## Object type aliases: creating subtypes when you need them

We've used abstract types a lot in this book. You'll notice that abstract types by themselves are generally made from very general concepts, such as `Person` and `HasNameAndCoffins`. In databases in real life you'll probably see them in the forms `HasEmail`, `HasID` and so on, which get extended to make subtypes.

Aliases will first look similar to extending an abstract type. Let's first compare the syntax between `extending` and using an alias so that you will be able to spot the difference:

```sdl
type Vampire extending Person {
    # Properties and links
}

alias AliasPerson := Person {
    # Computables, etc.
};
```

The first difference is that an alias uses `:=` instead of `extending`. In other words, an alias is a computed expression. Also note that `alias Vampire` ends in a semicolon - again, because it is an expression. And since aliases are expressions and not standalone types, they can't be inserted into a database. Instead, they point to data — often a type that exists in the database — and can give it an extra shape on top of the original. So you can query an alias, but you can't insert one.

Let's make an alias for fun for our schema too. Looking at the Demeter again, we see that the ship left from Varna in Bulgaria and reached London. We'll imagine in our game that we have built Varna up into a big port for the characters to explore, and are changing the schema to reflect this. Right now our `Crewman` type just looks like this:

```sdl
type Crewman extending HasNumber, Person {
  overloaded name: str {
    default := 'Crewman ' ++ <str>.number;
  }
}
```

Imagine that we would like a `CrewmanInBulgaria` type as well, because Bulgarians use the term 'Gospodin' to be polite (Bulgarian for "Mister") and our game would like to reflect that. A Crewman will be called "Gospodin (name)" whenever they are in Bulgaria. In addition, the fresh Bulgarian air gives sailors extra strength in our game so our `CrewmanInBulgaria` objects will also be a bit stronger. You can see that an alias is much better than an entirely separate type here, because all we want to do is have a different shape to work with when making queries. Here's how to do that:

```sdl
alias CrewmanInBulgaria := Crewman {
  name := 'Gospodin ' ++ .name,
  strength := .strength + <int16>1,
  # Just in case we want to filter on the original name
  original_name := .name,
};
```

You'll notice right away that `name` and `strength` inside the alias are separated by commas, not semicolons. That's a clue that this isn't creating a new type: it's just creating a _shape_ on top of the existing `Crewman` type.

Let's do a schema migration and then take a look at the error we get if we try to insert a `CrewmanInBulgaria`. Remember, it won't work because an alias is just an expression, not a type that can be inserted:

```edgeql
insert CrewmanInBulgaria {name := "New Crewman", number := 6};
```

Here is the error:

```
error: cannot insert into expression alias 'default::CrewmanInBulgaria'
```

So all inserts are still done through the `Crewman` type. But because an alias is a subtype and a shape, we can select it in the same way as anything else. Let's now compare a `select` on the `Crewman` objects to a `select` with the `CrewmanInBulgaria` alias:

```
db> select Crewman { name, strength };
{
  default::Crewman {name: 'Crewman 1', strength: 1},
  default::Crewman {name: 'Crewman 2', strength: 2},
  default::Crewman {name: 'Crewman 3', strength: 1},
  default::Crewman {name: 'Crewman 4', strength: 2},
  default::Crewman {name: 'Crewman 5', strength: 5},
}
db> select CrewmanInBulgaria { original_name, name, strength };
{
  default::Crewman {original_name: 'Crewman 1', name: 'Gospodin Crewman 1', strength: 2},
  default::Crewman {original_name: 'Crewman 2', name: 'Gospodin Crewman 2', strength: 3},
  default::Crewman {original_name: 'Crewman 3', name: 'Gospodin Crewman 3', strength: 2},
  default::Crewman {original_name: 'Crewman 4', name: 'Gospodin Crewman 4', strength: 3},
  default::Crewman {original_name: 'Crewman 5', name: 'Gospodin Crewman 5', strength: 6},
}
```

The expression works well, giving names that start with Gospodin and strength values a bit higher than outside of Bulgaria. But note that the expression still returns a `default::Crewman`, as the alias is just an expression on top of the original type.

## Local expression aliases: creating new names for types in a query

It's somewhat interesting that our alias is just declared using a `:=` when we wrote `alias CrewmanInBulgaria := Crewman`. Would it be possible to do something similar inside a query? The answer is yes: we can use `with` and then give a new name for an existing type. (In fact, the keyword `with` that we have been using the whole time is defined as a "{ref}`block used to define aliases <docs:ref_eql_with>`").

Let's say that we want to compare the strengths of all our `MinorVampire` objects. This first query won't work the way we want it to, and you can probably guess why:

```edgeql
select MinorVampire.name ++ ' is stronger than ' ++ MinorVampire.name ++ '? ' 
++ <str>(MinorVampire.name > MinorVampire.name);
```

It doesn't work because we are simply comparing one object against itself every time.

```
{
  'Vampire Woman 1 is stronger than Vampire Woman 1? false',
  'Vampire Woman 2 is stronger than Vampire Woman 2? false',
  'Vampire Woman 3 is stronger than Vampire Woman 3? false',
  'Lucy is stronger than Lucy? false',
}
```

We know that the `detached` keyword can help by pulling up a separate set of objects for a type. It won't help us here though unfortunately as we have to use it twice: once to concatenate the names, and again to compare strengths:

```edgeql
select MinorVampire.name ++ ' is stronger than ' ++ detached MinorVampire.name ++ '? '
++ <str>(MinorVampire.name > detached MinorVampire.name);
```

Just a small portion of the output shows what the problem is here: using `detached` pulls up a separate set of `MinorVampire` objects each time. Every time `detached` is used, the number of objects doubles!

```
{
  'Vampire Woman 1 is stronger than Vampire Woman 1? false',
  'Vampire Woman 1 is stronger than Vampire Woman 1? false',
  'Vampire Woman 1 is stronger than Vampire Woman 1? false',
  'Vampire Woman 1 is stronger than Vampire Woman 1? true',
  'Vampire Woman 1 is stronger than Vampire Woman 2? false',
  'Vampire Woman 1 is stronger than Vampire Woman 2? false',
  'Vampire Woman 1 is stronger than Vampire Woman 2? false',
}
```

Instead, we can use an alias for a set of `MinorVampire` objects and use that to compare:

```edgeql
with M := MinorVampire,
select M.name ++ ' is stronger than ' ++ MinorVampire.name ++ '? ' ++ 
<str>(M.strength > MinorVampire.strength);
```

The output is now closer to what we want, except that some objects with the same values are still being compared with each other. Here is part of the output (it's still pretty long):

```
{
  'Vampire Woman 1 is stronger than Vampire Woman 1? false',
  'Vampire Woman 1 is stronger than Vampire Woman 2? false',
  'Vampire Woman 1 is stronger than Vampire Woman 3? false',
  'Vampire Woman 1 is stronger than Lucy? false',
  'Vampire Woman 2 is stronger than Vampire Woman 1? true',
  'Vampire Woman 2 is stronger than Vampire Woman 2? false',
  'Vampire Woman 2 is stronger than Vampire Woman 3? false',
  'Vampire Woman 2 is stronger than Lucy? false',
}
```

And since we are using `M` as an expression alias, we can use it to filter. Let's filter out the objects that share the same id.

```edgeql
with M := MinorVampire,
select M.name ++ ' is stronger than ' ++ MinorVampire.name ++ '? ' ++ 
<str>(M.strength > MinorVampire.strength) filter M.id != MinorVampire.id;
```

And now we no longer have any duplicate names.

```
{
  'Vampire Woman 1 is stronger than Vampire Woman 2? false',
  'Vampire Woman 1 is stronger than Vampire Woman 3? false',
  'Vampire Woman 1 is stronger than Lucy? false',
  'Vampire Woman 2 is stronger than Vampire Woman 1? true',
  'Vampire Woman 2 is stronger than Vampire Woman 3? false',
  'Vampire Woman 2 is stronger than Lucy? false',
  'Vampire Woman 3 is stronger than Vampire Woman 1? true',
  'Vampire Woman 3 is stronger than Vampire Woman 2? true',
  'Vampire Woman 3 is stronger than Lucy? false',
  'Lucy is stronger than Vampire Woman 1? true',
  'Lucy is stronger than Vampire Woman 2? true',
  'Lucy is stronger than Vampire Woman 3? false',
}
```

But we aren't limited to just writing `:= MinorVampire` either. Because an alias is simply an expression, we can make some modifications to the set of objects we have been calling `M`. Let's make a set of `MinorVampire` objects (an "expression alias" of `MinorVampire`) that have a bit more strength than the regular set and compare them to the regular set of `MinorVampire` objects. While we're at it, let's change their names a bit too. This is starting to feel a lot like the alias we have in our schema, don't you think? The query now looks like this:

```edgeql
with PumpedUp := MinorVampire {
name := 'Pumped up ' ++ .name,
strength := .strength + <int16>2
 },
select PumpedUp.name ++ ' is stronger than ' ++ MinorVampire.name ++
'? ' ++ 
<str>(PumpedUp.strength > MinorVampire.strength) filter PumpedUp.id != MinorVampire.id;
```

Now the expression returns a lot more `true`, because the chance of having greater strength than the other object is that much greater.

```
{
  'Pumped up Vampire Woman 1 is stronger than Vampire Woman 2? true',
  'Pumped up Vampire Woman 1 is stronger than Vampire Woman 3? false',
  'Pumped up Vampire Woman 1 is stronger than Lucy? false',
  'Pumped up Vampire Woman 2 is stronger than Vampire Woman 1? true',
  'Pumped up Vampire Woman 2 is stronger than Vampire Woman 3? true',
  'Pumped up Vampire Woman 2 is stronger than Lucy? true',
  'Pumped up Vampire Woman 3 is stronger than Vampire Woman 1? true',
  'Pumped up Vampire Woman 3 is stronger than Vampire Woman 2? true',
  'Pumped up Vampire Woman 3 is stronger than Lucy? true',
  'Pumped up Lucy is stronger than Vampire Woman 1? true',
  'Pumped up Lucy is stronger than Vampire Woman 2? true',
  'Pumped up Lucy is stronger than Vampire Woman 3? true',
}
```

So how about adding links and properties in the same way that we did to our `CrewmanInBulgaria` alias? We can do that too by using `select` and then adding any new links and properties you want inside `{}`. Here's a simple example:

```edgeql
with NPCExtraInfo := NPC {
    would_win_against_dracula := .strength > Vampire.strength
  }
  select NPCExtraInfo {
    name,
    would_win_against_dracula
  };
```

And here's the result. Looks like nobody wins:

```
{
  default::NPC {name: 'Jonathan Harker', would_win_against_dracula: {false}},
  default::NPC {name: 'The innkeeper', would_win_against_dracula: {false}},
  default::NPC {name: 'Mina Murray', would_win_against_dracula: {false}},
  default::NPC {name: 'John Seward', would_win_against_dracula: {false}},
  default::NPC {name: 'Quincey Morris', would_win_against_dracula: {false}},
  default::NPC {name: 'Arthur Holmwood', would_win_against_dracula: {false}},
  default::NPC {name: 'Abraham Van Helsing', would_win_against_dracula: {false}},
  default::NPC {name: 'Lucy Westenra', would_win_against_dracula: {false}},
  default::NPC {name: 'Renfield', would_win_against_dracula: {false}},
}
```

Finally, let's create a quick type alias where Dracula has achieved all his goals and now rules London. We can give it the alias `DraculaKingOfLondon`, and a link to `subjects` (people who live under a king) that will be every `Person` that has been to London. Then we'll select this type, and also count how many subjects there are. It looks like this:

```edgeql
with DraculaKingOfLondon := Vampire {
      name := .name ++ ', King of London',
      subjects := (select Person filter 'London' in .places_visited.name),
    } 
select DraculaKingOfLondon {
  name,
  subjects: {name},
  number_of_subjects := count(.subjects)
} filter .name = 'Count Dracula';
```

Here's the output:

```
{
  default::Vampire {
    name: 'Count Dracula, King of London',
    subjects: {
      default::NPC {name: 'Jonathan Harker'},
      default::NPC {name: 'Mina Murray'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
      default::NPC {name: 'Abraham Van Helsing'},
      default::NPC {name: 'Lucy Westenra'},
      default::NPC {name: 'Renfield'},
      default::NPC {name: 'Lord Billy'},
    },
    number_of_subjects: 9,
  },
}
```

Note that this expression works because Count Dracula is our only `Vampire` object: he's almost like a unique global object. If we had more `Vampire` objects in the database we would need to filter by `name`.

## Other types of aliases

Aliases can be used in a number of other ways too. So far we have used object type aliases, but an alias is just an expression which makes them really quite open-ended. An alias can be a single scalar, a set of scalars, a tuple or set of tuples, a query, and so on. In this case an alias can feel a bit like a global value because it gives us quick access to some data that otherwise would be stored in a format like JSON somewhere.

For example, we could take an expression that helps us keep track of all the names we have in our database and turn it into an alias for convenience. At the moment, we have names for every `HasNameAndCoffins` object, plus every `modern_name` inside `Place`, `name` inside `Landmark` and of course the `name` property for the `Person` type:

```
alias AllNames := (
  distinct (HasNameAndCoffins.name union
  Place.modern_name union
  Landmark.name union 
  Person.name)
);
```

Another alias we could put together is a tuple that holds all of the metadata for our game. This has a sort of JSON-like feel but is baked into our schema as an EdgeDB tuple. You could imagine this alias used for the player menu which displays differently depending on the language of the user:

```sdl
alias GameInfo := (
  title := ( 
    en := "Dracula the Immortal",
    fr := "Dracula l'immortel",
    no := "Dracula den udødelige",
    ro := "Dracula, nemuritorul"
  ),
  country := "Norway",
  date_published := 2023,
  website := "www.draculatheimmortal.com"
);
```

After doing a migration you can see how easy and readable these queries become:

```
db> select GameInfo.title.ro;
{'Dracula, nemuritorul'}
db> select 'Max Demian' in AllNames;
{true}
db> select 'Canada' in AllNames;
{false}
```

The {ref}`documentation on aliases <docs:ref_cheatsheet_aliases>` has a number of other interesting examples of how you might want to use an alias in your project.

## Mutation rewrites

Since version 3.0, EdgeDB allows us to automatically rewrite properties of an object whenever an insert or update happens. The syntax for a mutation rewrite is really simple: just choose `insert` and/or `update`, and then add the expression.

```
rewrite {insert | update} [, ...]
  using expr
```

A mutation rewrite makes it really easy to keep track of when an object was last updated. To do this, just add a property to the object and a mutation rewrite that calls `datetime_of_statement()` whenever the object is inserted or updated. Let's give this a try on our `PC` type:

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement()
  }
  required number: PCNumber {
    default := sequence_next(introspect PCNumber);
  }
  last_updated: datetime {
    rewrite insert, update using (datetime_of_statement());
  }
}
```

But while we are at it, let's stretch our imagination a bit and make another mutation rewrite for fun for the `PC` type. Imagine that every time a `PC` is created or gets to a save point (which updates it) we will give it a chance to win a bonus item at the same time. Let's call it a `LotteryTicket` and add some items that are useful to vampire hunters.

```sdl
scalar type LotteryTicket extending enum <Nothing, WallChicken, 
ChainWhip, Crucifix, Garlic>;
```

Most of the time a lottery ticket will be nothing, but sometimes it will be a bonus item. Let's make a function to represent that:

```sdl
  function get_ticket() -> LotteryTicket using (
    with rnd := <int16>(random() * 10),
    select(LotteryTicket.Nothing if rnd <= 6 else
    LotteryTicket.WallChicken    if rnd = 7 else
    LotteryTicket.ChainWhip      if rnd = 8 else
    LotteryTicket.Crucifix       if rnd = 9 else
    LotteryTicket.Garlic)
  );
```

Putting all those together, here are the changes to make to the schema:

```sdl
type PC extending Person {
  required class: Class;
  required created_at: datetime {
    default := datetime_of_statement()
  }
  required number: PCNumber {
    default := sequence_next(introspect PCNumber);
  }
  last_updated: datetime {
    rewrite insert, update using (datetime_of_statement());
  }
  bonus_item: LotteryTicket {
    rewrite insert, update using (get_ticket());
  }
}

scalar type LotteryTicket extending enum <Nothing, WallChicken, ChainWhip, Crucifix, Garlic>;

function get_ticket() -> LotteryTicket using (
  with rnd := <int16>(random() * 10),
  select(LotteryTicket.Nothing if rnd <= 6 else
  LotteryTicket.WallChicken if rnd = 7 else
  LotteryTicket.ChainWhip if rnd = 8 else
  LotteryTicket.Crucifix if rnd = 9 else
  LotteryTicket.Garlic)
);
```

Once the migration is done, let's insert a new `PC` and see what she gets! First the insert:

```edgeql
insert PC {
 name := 'Sypha',
 class := Class.Mystic
};
```

And now a few queries to see what Sypha's bonus item is. This will return something different every time, so let's "update" her by...just giving her the same name as before.

```edgeql
update PC filter .name = 'Sypha' set { name := .name };
```

This will still count as an update to the object and thus the `last_updated` property will change, while the `bonus_item` is _likely_ to change depending on the random number chosen. The output will look something like this:

```
db> select PC { name, class, bonus_item, last_updated } filter .name = 'Sypha';
{
  default::PC {
    name: 'Sypha',
    class: Mystic,
    bonus_item: Nothing,
    last_updated: <datetime>'2023-06-18T07:50:56.181567Z',
  },
}
db> update PC filter .name = 'Sypha' set { name := .name };
{default::PC {id: 803d4486-0dac-11ee-9250-4746f54d7008}}
db> select PC { name, class, bonus_item, last_updated } filter .name = 'Sypha';
{
  default::PC {
    name: 'Sypha',
    class: Mystic,
    bonus_item: Crucifix,
    last_updated: <datetime>'2023-06-18T07:51:03.998048Z',
  },
}
```

[Here is all our code so far up to Chapter 17.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you display every NPC's name, strength, name and population of cities visited, and age (displaying 0 if age = `{}`)? Try it on a single line.

2. The query in 1. showed a lot of numbers without any context. What should we do?

3. How would you create an alias that contains all the months of the year?

4. How do you make sure that no data is lost when changing a type's properties from owned properties to properties extended from abstract types?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan the detective._
