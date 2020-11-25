# Chapter 17 - Poor Renfield. Poor Mina.

> Last chapter Dr. Seward and Dr. Van Helsing wanted to let Renfield out, but couldn't trust him. But it turns out that Renfield was telling the truth! That night, Dracula found out that they were destroying his coffins and decided to attack Mina. He succeeded, and now Mina is slowly turning into a vampire. She is still human, but has a connection with Dracula now. 

> The group finds Renfield in a pool of blood, dying. Renfield is sorry and tells them the truth. He was in communication with Dracula and thought that he would help him become a vampire too, so he let him in the house. But once inside Dracula ignored him, and headed for Mina's room. Renfield attacked Dracula to try to stop him from hurting her, but Dracula was much stronger and won.

> Van Helsing does not give up though, and has a good idea. If Mina is now connected to Dracula, what happens if he uses hypnotism on her? Could that work? He takes out his pocket watch and tells her: "Please concentrate on this watch. You are beginning to feel sleepy...what do you feel? Think about the man who attacked you, try to feel where he is..."


## Named tuples

Remember the function `fight()` that we made? It was overloaded to take either `(Person, Person)` or `(str, Person)` as input. Let's give it Dracula and Renfield:

```
WITH 
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),
  renfield := (SELECT Person FILTER .name = 'Renfield'),
SELECT fight(dracula, renfield);
```

The output is of course `{'Count Dracula wins!'}`. No surprise there.

One other way to do the same query is with a single tuple. Then we can give it to the function with `.0` for Dracula and `.1` for Renfield:

```
WITH fighters := (
  (SELECT Person FILTER .name = 'Count Dracula'), (SELECT Person FILTER .name = 'Renfield')
  ),
  SELECT fight(fighters.0, fighters.1);
```

That's not bad, but there is a way to make it clearer: we can give names to the items in the tuple instead of using `.0` and `.1`. It looks like a regular computable, using `:=`:

```
WITH fighters := (
  dracula := (SELECT Person FILTER .name = 'Count Dracula'), 
  renfield := (SELECT Person FILTER .name = 'Renfield')),
  SELECT fight(fighters.dracula, fighters.renfield);
```

Here's one more example of a named tuple:

```
WITH minor_vampires := (
  women := (SELECT MinorVampire FILTER .name LIKE '%Woman%'), 
  lucy := (SELECT MinorVampire FILTER .name LIKE '%Lucy%')),
SELECT (minor_vampires.women.name, minor_vampires.lucy.name);
```

The output is:

```
{('Woman 1', 'Lucy Westenra'), ('Woman 2', 'Lucy Westenra'), ('Woman 3', 'Lucy Westenra')}
```

Renfield is no longer alive, so we need to use `UPDATE` to give him a `last_appearance`. Let's do a fancy one again where we `SELECT` the update we just made and display that information:

```
SELECT ( # Put the whole update inside
  UPDATE NPC filter .name = 'Renfield'
    SET {
  last_appearance := <cal::local_date>'1887-10-03'
}) # then use it to call up name and last_appearance
  {
  name, 
  last_appearance
  };
```

This gives us: `{default::NPC {name: 'Renfield', last_appearance: <cal::local_date>'1887-10-03'}}`

## Putting abstract types together

Wherever there are vampires, there are vampire hunters. Sometimes they will destroy their coffins, and other times vampires will build more. So it would be cool to create a quick function called `change_coffins()` to change the number of coffins in a place. With this function we could write something like `change_coffins('London', -13)` to reduce it by 13, for example. But the problem right now is this: 

- the `HasCoffins` type is an abstract type, with one property: `coffins`
- places that can have coffins are `Place` and all the types from it, plus `Ship`,
- the best way to filter is by `.name`, but `HasCoffins` doesn't have this property.

So maybe we can turn this type into something else called `HasNameAndCoffins`, and put the `name` and `coffins` properties inside there. This won't be a problem because every place needs a name and a number of coffins in our game. Remember, 0 coffins means that vampires can't stay in a place for long: just quick trips in at night before the sun rises.

Here is the type with its new property. We'll give it two constraints: `exclusive` and `max_len_value` to keep names from being too long.

```
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

```
    type Ship extending HasNameAndCoffins {
        multi link sailors -> Sailor;
        multi link crew -> Crewman;
    }
```

And the `Place` type. It's much simpler now.

```
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

Finally, we can change our `can_enter()` function. This one needed a `HasCoffins` type before: 

```
    function can_enter(person_name: str, place: HasCoffins) -> str
      using (
        with vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
        SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
            );   
```

But now that `HasNameAndCoffins` holds `name`, the user can now just enter a string. We'll change it to this:

```
function can_enter(person_name: str, place: str) -> str
  using (
  with 
    vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
    place := (SELECT HasNameAndCoffins FILTER .name = place LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
    );   
```

And now we can just enter `can_enter('Count Dracula', 'Munich')` to get `'Count Dracula cannot enter.'`. That makes sense: Dracula didn't bring any coffins there.

Finally, we can make our `change_coffins` function. It's easy:

```
    function change_coffins(place_name: str, number: int16) -> HasNameAndCoffins
       using (
         UPDATE HasNameAndCoffins FILTER .name = place_name
         SET {
           coffins := .coffins + number
         }
      );
```

Now let's give the ship `The Demeter` some coffins.

```
SELECT change_coffins('The Demeter', 10);
```

Then we'll make sure that it got them:

```
 SELECT Ship {
  name,
  coffins,
};
```

We get: `{default::Ship {name: 'The Demeter', coffins: 10}}`. The Demeter got its coffins!

[Here is all our code so far up to Chapter 17.](code.md)

## Time to practice

1. How would you display every NPC's name, strength, name and population of cities visited, and age (displaying 0 if age = `{}`)? Try it on a single line.

2. The query in 1. showed a lot of numbers without any context. What should we do?

3. Renfield is now dead and needs a `last_appearance`. Try writing a function called `make_dead(person_name: str, date: str) ->  Person` that lets you just write the character name and date to do it.

[See the answers here.](answers.md)

Up next in Chapter 18: [Jonathan the detective.](../chapter18/index.md)
