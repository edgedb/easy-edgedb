# Chapter 17 - Poor Renfield. Poor Mina.

> Last chapter Dr. Seward and Dr. Van Helsing wanted to let Renfield out, but couldn't trust him. But it turns out that Renfield was telling the truth! That night, Dracula found out that they were destroying his coffins and decided to attack Mina. He succeeded, and now Mina is slowly turning into a vampire. She is still human, but has a connection with Dracula now.

> The group finds Renfield in a pool of blood, dying. Renfield is sorry and tells them the truth. He was in communication with Dracula and thought that he would help him become a vampire too, so he let him in the house. But once inside Dracula ignored him, and headed for Mina's room. Renfield attacked Dracula to try to stop him from hurting her, but Dracula was much stronger and won.

> Mina does not give up though, and has a good idea. If she is now connected to Dracula, what happens if Van Helsing uses hypnotism on her? Could that work? He takes out his pocket watch and tells her: "Please concentrate on this watch. You are beginning to feel sleepy...what do you feel? Think about the man who attacked you, try to feel where he is..."

## Named tuples

Remember the function `fight()` that we made? It was overloaded to take either `(Person, Person)` or `(str, Person)` as input. Let's give it Dracula and Renfield:

```edgeql
WITH
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),
  renfield := (SELECT Person FILTER .name = 'Renfield'),
SELECT fight(dracula, renfield);
```

The output is of course `{'Count Dracula wins!'}`. No surprise there.

One other way to do the same query is with a single tuple. Then we can give it to the function with `.0` for Dracula and `.1` for Renfield:

```edgeql
WITH fighters := (
    (SELECT Person FILTER .name = 'Count Dracula'),
    (SELECT Person FILTER .name = 'Renfield')
  ),
SELECT fight(fighters.0, fighters.1);
```

That's not bad, but there is a way to make it clearer: we can give names to the items in the tuple instead of using `.0` and `.1`. It looks like a regular computable, using `:=`:

```edgeql
WITH fighters := (
    dracula := (SELECT Person FILTER .name = 'Count Dracula'),
    renfield := (SELECT Person FILTER .name = 'Renfield')
  ),
SELECT fight(fighters.dracula, fighters.renfield);
```

Here's one more example of a named tuple:

```edgeql
WITH minor_vampires := (
    women := (SELECT MinorVampire FILTER .name LIKE '%Woman%'),
    lucy := (SELECT MinorVampire FILTER .name LIKE '%Lucy%')
  ),
SELECT (minor_vampires.women.name, minor_vampires.lucy.name);
```

The output is:

```
{('Woman 1', 'Lucy Westenra'), ('Woman 2', 'Lucy Westenra'), ('Woman 3', 'Lucy Westenra')}
```

Renfield is no longer alive, so we need to use `UPDATE` to give him a `last_appearance`. Let's do a fancy one again where we `SELECT` the update we just made and display that information:

```edgeql
SELECT ( # Put the whole update inside
  UPDATE NPC filter .name = 'Renfield'
  SET {
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
SELECT ('Lucy Westenra', 'Renfield') = (character1 := 'Lucy Westenra', character2 := 'Renfield');
```

will return `{true}`.

## Putting abstract types together

Wherever there are vampires, there are vampire hunters. Sometimes they will destroy their coffins, and other times vampires will build more. So it would be cool to create a quick function called `change_coffins()` to change the number of coffins in a place. With this function we could write something like `change_coffins('London', -13)` to reduce it by 13, for example. But the problem right now is this:

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
    constraint exclusive;
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
function can_enter(person_name: str, place: HasCoffins) -> str
  using (
    with vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

But now that `HasNameAndCoffins` holds `name`, the user can now just enter a string. We'll change it to this:

```sdl
function can_enter(person_name: str, place: str) -> str
  using (
    with
      vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
      enter_place := (SELECT HasNameAndCoffins FILTER .name = place LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF enter_place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

And now we can just enter `can_enter('Count Dracula', 'Munich')` to get `'Count Dracula cannot enter.'`. That makes sense: Dracula didn't bring any coffins there.

Finally, we can make our `change_coffins` function. It's easy:

```sdl
function change_coffins(place_name: str, number: int16) -> HasNameAndCoffins
  using (
    UPDATE HasNameAndCoffins FILTER .name = place_name
    SET {
      coffins := .coffins + number
    }
  );
```

Now let's give the ship `The Demeter` some coffins.

```edgeql
SELECT change_coffins('The Demeter', 10);
```

Then we'll make sure that it got them:

```edgeql
SELECT Ship {
  name,
  coffins,
};
```

We get: `{default::Ship {name: 'The Demeter', coffins: 10}}`. The Demeter got its coffins!

## Aliases: creating subtypes when you need them

We've used abstract types a lot in this book. You'll notice that abstract types by themselves are generally made from very general concepts: `Person`, `HasNameAndCoffins`, etc. In databases in real life you'll probably see them in the forms `HasEmail`, `HasID` and so on, which get extended to make subtypes. Aliases also make subtypes, except they use `:=` instead of `extending` and draw from full types.

Let's make an alias for our schema too. Looking at the Demeter again, the ship left from Varna in Bulgaria and reached London. We'll imagine in our game that we have built Varna up into a big port for the characters to explore, and are changing the schema to reflect this. Right now our `Crewman` type just looks like this:

```sdl
type Crewman extending HasNumber, Person {
}
```

Imagine that for some reason we would like a `CrewmanInBulgaria` alias as well, because Bulgarians call each other 'Gospodin' instead of Mr. and our game needs to reflect that. Our Crewman types will get called "Gospodin (name)" whenever they are there. Let's also add a `current_location` computable that makes a link to `Place` types with the name Bulgaria. Here's how to do that:

```sdl
alias CrewmanInBulgaria := Crewman {
  name := 'Gospodin ' ++ .name,
  current_location := (SELECT Place filter .name = 'Bulgaria'),
}
```

You'll notice right away that `name` and `current_location` inside the alias are separated by commas, not semicolons. That's a clue that this isn't creating a new type: it's just creating a _shape_ on top of the existing `Crewman` type. For the same reason, you can't do an `INSERT CrewmanInBulgaria`, because there is no such type. It gives this error:

```
error: cannot insert into expression alias 'default::CrewmanInBulgaria'
```

So all inserts are still done through the `Crewman` type. But because an alias is a subtype and a shape, we can select it in the same way as anything else. Let's add Bulgaria now,

```edgeql
INSERT Country {
  name := 'Bulgaria'
};
```

And then select this alias to see what we get:

```edgeql
SELECT CrewmanInBulgaria {
  name,
  current_location: {
    name
  }
};
```

And now we see the same `Crewman` types under their `CrewmanInBulgaria` alias: with _Gospodin_ added to their name and linked to the `Country` type we just inserted.

```
{
  default::Crewman {
    name: 'Gospodin Crewman 0',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 1',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 2',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 3',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 4',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 5',
    current_location: default::Country {name: 'Bulgaria'},
  },
}
```

The [documentation on aliases](https://www.edgedb.com/docs/cheatsheet/aliases/) mentions that they let you use "the full power of EdgeQL (expressions, aggregate functions, backwards link navigation) from GraphQL", so keep aliases in mind if you use GraphQL a lot.

## Creating new names for types in a query (local expression aliases)

It's somewhat interesting that our alias is just declared using a `:=` when we wrote `alias CrewmanInBulgaria := Crewman`. Would it be possible to do something similar inside a query? The answer is yes: we can use `WITH` and then give a new name for an existing type. (In fact, the keyword `WITH` that we have been using the whole time is defined as a "[block used to define aliases](https://www.edgedb.com/docs/edgeql/statements/with)"). Take a simple query like this that shows Count Dracula and the names of his slaves:

```edgeql
SELECT Vampire {
  name,
  slaves: {
    name
  }
};
```

If we wanted to use `WITH` to create a new type that is identical to `Vampire`, we would just do this.

```edgeql
WITH Drac := Vampire,
SELECT Drac {
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
      default::MinorVampire {name: 'Lucy Westenra'},
    },
  },
}
```

But where it becomes useful is when using this new type to perform operations or comparisons on the type it is created from. It's the same as `DETACHED`, but we give it a name and have some more flexibility to use it.

So let's give this a try. We'll pretend that we are testing out our game engine. Right now our `fight()` function is ridiculously simple, but let's pretend that it's complicated and needs a lot of testing. Here's the variant we would use:

```sdl
function fight(one: Person, two: Person) -> str
  using (
    SELECT one.name ++ ' wins!' IF one.strength > two.strength ELSE two.name ++ ' wins!'
  );
```

But for debugging purposes it would be nice to have some more info. Let's create the same function but call it `fight_2()` and add some more information on who is fighting who.

```edgeql
CREATE FUNCTION fight_2(one: Person, two: Person) -> str
  USING (
    SELECT one.name ++ ' fights ' ++ two.name ++ '. ' ++ one.name ++ ' wins!'
      IF one.strength > two.strength
      ELSE one.name ++ ' fights ' ++ two.name ++ '. ' ++ two.name ++ ' wins!'
  );
```

So let's make our `MinorVampire` types fight each other and see what output we get. We have four of them (the three vampire women plus Lucy). First let's just put the `MinorVampire` type into the function and see what we get. Try to imagine what the output will be.

```edgeql
SELECT fight_2(MinorVampire, MinorVampire);
```

So the output for this is...

...

```
{
  'Lucy Westenra fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

The function only gets used four times because a set with only a single object goes in every time...and each `MinorVampire` is fighting itself. That's probably not what we wanted. Now let's try our local type alias again.

```edgeql
WITH M := MinorVampire,
SELECT fight_2(M, MinorVampire);
```

By the way, so far this is exactly the same as:

```edgeql
SELECT fight_2(MinorVampire, DETACHED MinorVampire);
```

The output is too long now:

```
{
  'Lucy Westenra fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 2 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 3 fights Lucy Westenra. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 1. Lucy Westenra wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Lucy Westenra fights Woman 2. Lucy Westenra wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
  'Lucy Westenra fights Woman 3. Lucy Westenra wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

We succeeded at getting each `MinorVampire` type to fight the other one, but there are still `MinorVampire`s fighting themselves (Lucy vs. Lucy, Woman 1 vs. Woman 1, etc.). This is where the convenience of the local type alias comes in: we can filter on it, for example. Now we'll filter to only use `fight_2()` with objects that are not identical to each other:

```edgeql
WITH M := MinorVampire,
SELECT fight_2(M, MinorVampire) FILTER M != MinorVampire;
```

And now we finally have every combination of `MinorVampire` fighting the other one, with no duplicates.

```
{
  'Lucy Westenra fights Woman 1. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 2. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 3. Lucy Westenra wins!',
  'Woman 1 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
}
```

Perfect!

With just `DETACHED` this wouldn't work: `SELECT fight_2(MinorVampire, DETACHED MinorVampire) FILTER MinorVampire != DETACHED MinorVampire;` won't do it because the first `DETACHED MinorVampire` isn't a variable name. Without that name to access, the next `DETACHED MinorVampire` is just a new `DETACHED MinorVampire` with no relation to the other one.

So how about adding links and properties in the same way that we did to our `CrewmanInBulgaria` alias? We can do that too by using `SELECT` and then adding any new links and properties you want inside `{}`. Here's a simple example:

```edgeql
WITH NPCExtraInfo := (
    SELECT NPC {
      would_win_against_dracula := .strength > Vampire.strength
    }
  )
SELECT NPCExtraInfo {
  name,
  would_win_against_dracula
};
```

And here's the result. Looks like nobody wins:

```
{
  default::NPC {name: 'Jonathan Harker', would_win_against_dracula: {false}},
  default::NPC {name: 'Renfield', would_win_against_dracula: {false}},
  default::NPC {name: 'The innkeeper', would_win_against_dracula: {false}},
  default::NPC {name: 'Mina Murray', would_win_against_dracula: {false}},
  default::NPC {name: 'John Seward', would_win_against_dracula: {false}},
  default::NPC {name: 'Quincey Morris', would_win_against_dracula: {false}},
  default::NPC {name: 'Arthur Holmwood', would_win_against_dracula: {false}},
  default::NPC {name: 'Abraham Van Helsing', would_win_against_dracula: {false}},
  default::NPC {name: 'Lucy Westenra', would_win_against_dracula: {false}},
}
```

Let's create a quick type alias where Dracula has achieved all his goals and now rules London. We'll create a quick new type called `DraculaKingOfLondon` with a better name, and a link to `subjects` (= people under a king) that will be every `Person` that has been to London. Then we'll select this type, and also count how many subjects there are. It looks like this:

```edgeql
WITH DraculaKingOfLondon := (
    SELECT Vampire {
      name := .name ++ ', King of London',
      subjects := (SELECT Person FILTER 'London' in .places_visited.name),
    }
  )
SELECT DraculaKingOfLondon {
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
      default::NPC {name: 'Renfield'},
      default::NPC {name: 'The innkeeper'},
      default::NPC {name: 'Mina Murray'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
      default::NPC {name: 'Abraham Van Helsing'},
      default::NPC {name: 'Lucy Westenra'},
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

3. Renfield is now dead and needs a `last_appearance`. Try writing a function called `make_dead(person_name: str, date: str) -> Person` that lets you just write the character name and date to do it.

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 18: [Jonathan the detective.](../chapter18/index.md)
