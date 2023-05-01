---
tags: Aliases, Named Tuples
---

# Chapter 17 - Poor Renfield. Poor Mina.

> Last chapter Dr. Seward and Dr. Van Helsing wanted to let Renfield out, but couldn't trust him. But it turns out that Renfield was telling the truth! That night, Dracula found out that the heroes were destroying his coffins and decided to attack Mina. Dracula succeeded, and now Mina is slowly turning into a vampire. She is still human, but has a connection with Dracula now.
>
> The group finds Renfield in a pool of blood, dying. Renfield is sorry and tells them the truth. Renfield had been in communication with Dracula because he wanted to become a vampire too, so he let Dracula into the house. But once inside Dracula ignored him, and headed for Mina's room. Renfield attacked Dracula to try to stop him from hurting Mina, but Dracula was much stronger and won.
>
> Mina has not given up though, and has a good idea. If she is now connected to Dracula, what happens if Van Helsing uses hypnotism on her? Could that work? He takes out his pocket watch and tells her: "Please concentrate on this watch. You are beginning to feel sleepy...what do you feel? Think about the man who attacked you, try to feel where he is..."

## Named tuples

Remember the function `fight()` that we made? It was overloaded to take either `(Person, Person)` or `(str, int16, str)` as input. Let's give it Dracula and Renfield:

```edgeql
with
  dracula := (select Person filter .name = 'Count Dracula'),
  renfield := (select Person filter .name = 'Renfield'),
select fight(dracula, renfield);
```

The output is of course `{'Count Dracula wins!'}`. No surprise there.

One other way to do the same query is with a single tuple. Then we can give it to the function with `.0` for Dracula and `.1` for Renfield:

```edgeql
with fighters := (
    (select Person filter .name = 'Count Dracula'),
    (select Person filter .name = 'Renfield')
  ),
select fight(fighters.0, fighters.1);
```

That's not bad, but there is a way to make it clearer: we can give names to the items in the tuple instead of using `.0` and `.1`. It looks like a regular computed link or property, using `:=`:

```edgeql
with fighters := (
    dracula := (select Person filter .name = 'Count Dracula'),
    renfield := (select Person filter .name = 'Renfield')
  ),
select fight(fighters.dracula, fighters.renfield);
```

Here's one more example of a named tuple:

```edgeql
with minor_vampires := (
    women := (select MinorVampire filter .name like '%Woman%'),
    lucy := (select MinorVampire filter .name like '%Lucy%')
  ),
select (minor_vampires.women.name, minor_vampires.lucy.name);
```

The output is:

```
{('Woman 1', 'Lucy'), ('Woman 2', 'Lucy'), ('Woman 3', 'Lucy')}
```

Renfield is no longer alive, so we need to use `update` to give him a `last_appearance`. Let's do a fancy one again where we `select` the update we just made and display that information:

```edgeql
select ( # Put the whole update inside
  update NPC filter .name = 'Renfield'
  set {
    last_appearance := <cal::local_date>'1887-10-03'
  }
) # then use it to call up name and last_appearance
{
  name,
  last_appearance
};
```

This gives us: `{default::NPC {name: 'Renfield', last_appearance: <cal::local_date>'1887-10-03'}}`

One last thing: naming an item in a tuple doesn't have any effect on the items inside. So this:

```edgeql
select ('Lucy Westenra', 'Renfield') = (character1 := 'Lucy Westenra', character2 := 'Renfield');
```

will return `{true}`.

## Putting abstract types together

Wherever there are vampires, there are vampire hunters. Sometimes they will destroy their coffins, and other times vampires will build more. It would be nice to have a generic way to update this information. But the problem right now is this:

- the `HasCoffins` type is an abstract type, with one property: `coffins`
- places that can have coffins are `Place` and all the types from it, plus `Ship`,
- the best way to filter is by `.name`, but `HasCoffins` doesn't have this property.

So maybe we can turn this type into something else called `HasNameAndCoffins`, and put the `name` and `coffins` properties inside there. This won't be a problem because every place needs a name and a number of coffins in our game. Remember, 0 coffins means that vampires can't stay in a place for long: just quick trips in at night before the sun rises.

Here is the type with its new property. We'll give it two constraints: `exclusive` and `max_len_value` to keep names from being too long.

```sdl
abstract type HasNameAndCoffins {
  required property coffins -> int16 {
    default := 0;
  }
  required property name -> str {
    delegated constraint exclusive;
    constraint max_len_value(30);
  }
}
```

So now we can change our `Ship` type (notice that we removed `name`)

```sdl
type Ship extending HasNameAndCoffins {
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

And the `Place` type. It's much simpler now.

```sdl
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

Finally, we can change our `can_enter()` function. This one needed a `HasCoffins` type before:

```sdl
function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := (select Person filter .name = person_name),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.' if has_coffins else vampire.name ++ ' cannot enter.'
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
    select vampire.name ++ ' can enter.' if enter_place.coffins > 0 else vampire.name ++ ' cannot enter.'
  );
```

And now we can just enter `can_enter('Count Dracula', 'Munich')` to get `'Count Dracula cannot enter.'`. That makes sense: Dracula didn't bring any coffins there.

Finally, we can make our generic update for changing the number of coffins. It's easy:

```edgeql
update HasNameAndCoffins filter .name = <str>$place_name
set {
  coffins := .coffins + <int16>$number
}
```

Now let's give the ship `The Demeter` some coffins.

```edgeql-repl
edgedb> update HasNameAndCoffins filter .name = <str>$place_name
....... set {
.......   coffins := .coffins + <int16>$number
....... };
Parameter <str>$place_name: The Demeter
Parameter <int16>$number: 10
```

Then we'll make sure that it got them:

```edgeql
select Ship {
  name,
  coffins,
};
```

We get: `{default::Ship {name: 'The Demeter', coffins: 10}}`. The Demeter got its coffins!

## Aliases: creating subtypes when you need them

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

The first difference is that an alias uses `:=` instead of `extending`. In other words, an alias is a computed expression. Also note that `alias Vampire` ends in a semicolon - again, because it is an expression. And since aliases are expressions and not standalone types, they can't be inserted into a database. Instead, they point to data—usually a type that exists in the database—and can give it an extra shape on top of the original. You can query an alias, but you can't insert one.

One other difference is that `extending` can only be used on an `abstract type`, while an alias can be used on just about anything.

Let's make an alias for our schema too. Looking at the Demeter again, we see that the ship left from Varna in Bulgaria and reached London. We'll imagine in our game that we have built Varna up into a big port for the characters to explore, and are changing the schema to reflect this. Right now our `Crewman` type just looks like this:

```sdl
type Crewman extending HasNumber, Person {
}
```

Imagine that we would like a `CrewmanInBulgaria` type as well, because Bulgarians call each other 'Gospodin' (Bulgarian for "Mister") and our game would like to reflect that. A Crewman will be called "Gospodin (name)" whenever they are in Bulgaria. In addition, the fresh Bulgarian air gives sailors extra strength in our game so our `CrewmanInBulgaria` objects will also be a bit stronger. But an entirely separate type doesn't feel right here - we just want to have a slightly different shape to work with when making queries. Here's how to do that:

```sdl
alias CrewmanInBulgaria := Crewman {
  name := 'Gospodin ' ++ .name,
  strength := .strength + <int16>1,
  # Just in case we want to filter on the original name
  original_name := .name,
};
```

You'll notice right away that `name` and `strength` inside the alias are separated by commas, not semicolons. That's a clue that this isn't creating a new type: it's just creating a _shape_ on top of the existing `Crewman` type. Let's now take a look at the error we get if we try to insert a `CrewmanInBulgaria`. Remember, it won't work because an alias is just an expression, not a type that can be inserted:

```edgeql
insert CrewmanInBulgaria {name := "New Crewman", number := 6};
```

Here is the error:

```
error: cannot insert into expression alias 'default::CrewmanInBulgaria'
```

So all inserts are still done through the `Crewman` type. But because an alias is a subtype and a shape, we can select it in the same way as anything else. Let's now compare a `select` on the `Crewman` objects to a `select` with the `CrewmanInBulgaria` alias:

```edgeql
edgedb> select Crewman { name, strength };
{
  default::Crewman {name: 'Crewman 1', strength: 1},
  default::Crewman {name: 'Crewman 2', strength: 2},
  default::Crewman {name: 'Crewman 3', strength: 1},
  default::Crewman {name: 'Crewman 4', strength: 2},
  default::Crewman {name: 'Crewman 5', strength: 5},
}
edgedb> select CrewmanInBulgaria { original_name, name, strength };
{
  default::Crewman {original_name: 'Crewman 1', name: 'Gospodin Crewman 1', strength: 2},
  default::Crewman {original_name: 'Crewman 2', name: 'Gospodin Crewman 2', strength: 3},
  default::Crewman {original_name: 'Crewman 3', name: 'Gospodin Crewman 3', strength: 2},
  default::Crewman {original_name: 'Crewman 4', name: 'Gospodin Crewman 4', strength: 3},
  default::Crewman {original_name: 'Crewman 5', name: 'Gospodin Crewman 5', strength: 6},
}
```

The expression works well, giving names that start with Gospodin and strength values a bit higher than outside of Bulgaria. But note that the expression still returns a `default::Crewman`, as the alias is just an expression on top of the original type.

Aliases can be used in a number of other ways too, such as on scalar types. The {ref}`documentation on aliases <docs:ref_cheatsheet_aliases>` has a number of other interesting examples of how you might want to use an alias in your project.

## Creating new names for types in a query (local expression aliases)

It's somewhat interesting that our alias is just declared using a `:=` when we wrote `alias CrewmanInBulgaria := Crewman`. Would it be possible to do something similar inside a query? The answer is yes: we can use `with` and then give a new name for an existing type. (In fact, the keyword `with` that we have been using the whole time is defined as a "{ref}`block used to define aliases <docs:ref_eql_with>`"). Take a simple query like this that shows Count Dracula and the names of his slaves:

```edgeql
select Vampire {
  name,
  slaves: {
    name
  }
};
```

If we wanted to use `with` to create a new type that is identical to `Vampire`, we would just do this.

```edgeql
with Drac := Vampire,
select Drac {
  name,
  slaves: {
    name
  }
};
```

So far this is nothing special, because the output is the same:

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Woman 1'},
      default::MinorVampire {name: 'Woman 2'},
      default::MinorVampire {name: 'Woman 3'},
      default::MinorVampire {name: 'Lucy'},
    },
  },
}
```

But where it becomes useful is when using this new type to perform operations or comparisons on the type it is created from. It's the same as `detached`, but we give it a name and have some more flexibility to use it.

So let's give this a try. We'll pretend that we are testing out our game engine. Right now our `fight()` function is ridiculously simple, but let's pretend that it's complicated and needs a lot of testing. Here's the variant we would use:

```sdl
function fight(one: Person, two: Person) -> str
  using (
    one.name ++ ' wins!' if one.strength > two.strength else
    two.name ++ ' wins!'
  );
```

But for debugging purposes it would be nice to have some more info. Let's create the same function but call it `fight_2()` and add some more information on who is fighting who.

```edgeql
create function fight_2(one: Person, two: Person) -> str
  using (
    one.name ++ ' fights ' ++ two.name ++ '. ' ++ one.name ++ ' wins!'
    if one.strength > two.strength else
    one.name ++ ' fights ' ++ two.name ++ '. ' ++ two.name ++ ' wins!'
  );
```

So let's make our `MinorVampire` types fight each other and see what output we get. We have four of them (the three vampire women plus Lucy). Let's make sure that all vampires have strength 9 unless otherwise specified:

```edgeql
update MinorVampire filter not exists .strength
set {
  strength := 9
};
```

First let's just put the `MinorVampire` type into the function and see what we get. Try to imagine what the output will be.

```edgeql
select fight_2(MinorVampire, MinorVampire);
```

So the output for this is...

...

```
{
  'Lucy fights Lucy. Lucy wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

The function only gets used four times because a set with only a single object goes in every time...and each `MinorVampire` is fighting itself. That's probably not what we wanted. Now let's try our local type alias again.

```edgeql
with M := MinorVampire,
select fight_2(M, MinorVampire);
```

By the way, so far this is exactly the same as:

```edgeql
select fight_2(MinorVampire, detached MinorVampire);
```

The output is too long now:

```
{
  'Lucy fights Lucy. Lucy wins!',
  'Lucy fights Woman 1. Lucy wins!',
  'Lucy fights Woman 2. Lucy wins!',
  'Lucy fights Woman 3. Lucy wins!',
  'Woman 1 fights Lucy. Lucy wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Lucy. Lucy wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Lucy. Lucy wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

We succeeded at getting each `MinorVampire` type to fight the other one, but there are still `MinorVampire`s fighting themselves (Lucy vs. Lucy, Woman 1 vs. Woman 1, etc.). This is where the convenience of the local type alias comes in: we can filter on it, for example. Now we'll filter to only use `fight_2()` with objects that are not identical to each other:

```edgeql
with M := MinorVampire,
select fight_2(M, MinorVampire) filter M != MinorVampire;
```

And now we finally have every combination of `MinorVampire` fighting the other one, with no duplicates.

```
{
  'Lucy fights Woman 1. Lucy wins!',
  'Lucy fights Woman 2. Lucy wins!',
  'Lucy fights Woman 3. Lucy wins!',
  'Woman 1 fights Lucy. Lucy wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Lucy. Lucy wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Lucy. Lucy wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
}
```

Perfect!

With just `detached` this wouldn't work: `select fight_2(MinorVampire, detached MinorVampire) filter MinorVampire != detached MinorVampire;` won't do it because the first `detached MinorVampire` isn't a variable name. Without that name to access, the next `detached MinorVampire` is just a new `detached MinorVampire` with no relation to the other one.

So how about adding links and properties in the same way that we did to our `CrewmanInBulgaria` alias? We can do that too by using `select` and then adding any new links and properties you want inside `{}`. Here's a simple example:

```edgeql
with NPCExtraInfo := (
    select NPC {
      would_win_against_dracula := .strength > Vampire.strength
    }
  )
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

Let's create a quick type alias where Dracula has achieved all his goals and now rules London. We'll create a quick new type called `DraculaKingOfLondon` with a better name, and a link to `subjects` (= people under a king) that will be every `Person` that has been to London. Then we'll select this type, and also count how many subjects there are. It looks like this:

```edgeql
with DraculaKingOfLondon := (
    select Vampire {
      name := .name ++ ', King of London',
      subjects := (select Person filter 'London' in .places_visited.name),
    }
  )
select DraculaKingOfLondon {
  name,
  subjects: {name},
  number_of_subjects := count(.subjects)
};
```

Here's the output:

```
{
  default::Vampire {
    name: 'Count Dracula, King of London',
    subjects: {
      default::NPC {name: 'Jonathan Harker'},
      default::NPC {name: 'The innkeeper'},
      default::NPC {name: 'Mina Murray'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
      default::NPC {name: 'Abraham Van Helsing'},
      default::NPC {name: 'Lucy Westenra'},
      default::NPC {name: 'Renfield'},
    },
    number_of_subjects: 9,
  },
}
```

[Here is all our code so far up to Chapter 17.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you display every NPC's name, strength, name and population of cities visited, and age (displaying 0 if age = `{}`)? Try it on a single line.

2. The query in 1. showed a lot of numbers without any context. What should we do?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan the detective._
