# Chapter 7 - Jonathan finally "leaves" the castle

> Jonathan sneaks into Dracula's room during the day and sees him sleeping inside a coffin. Now he knows that he is a vampire. A few days later Count Dracula says that he will leave tomorrow. Jonathan thinks this is a chance, and asks to leave now. Dracula says, "Fine, if you wish..." and opens the door: but there are a lot of wolves outside, howling and making loud sounds. Dracula says, "You are free to leave! Goodbye!" But Jonathan knows that the wolves will kill him if he steps outside. He also knows that Dracula called the wolves, and asks him to please close the door. Dracula smiles and closes the door...he knows that Jonathan is trapped. Later, Jonathan hears Dracula tell the vampire women that they can have him tomorrow after he leaves. Dracula's friends take him away the next day (he is inside a coffin), and Jonathan is alone...and soon it will be night. All the doors are locked. He decides to climb out the window, because it is better to die by falling than to be alone with the vampire women. He writes "Good-bye, all! Mina!" in his journal and begins to climb the wall.

## More constraints

While Jonathan climbs the wall, we can continue to work on our database schema. In our book, no character has the same name so there should only be one Mina Murray, one Count Dracula, and so on. This is a good time to put a [constraint](https://edgedb.com/docs/datamodel/constraints#ref-datamodel-constraints) on `name` in the `Person` type to make sure that we don't have duplicate inserts. A `constraint` is a limitation, which we saw already in `age` for humans that can only go up to 120. For `name` we can give it another one called `constraint exclusive` which prevents two objects of the same type from having the same name. You can put a `constraint` in a block after the property, like this:

```
abstract type Person {
  required property name -> str { ## Add a block
      constraint exclusive;       ## and the constraint
  }
  multi link places_visited -> Place;
  link lover -> Person;
}
```

Now we know that there will only be one `Jonathan Harker`, one `Mina Murray`, and so on. In real life this is often useful for email addresses, User IDs, and other properties that you always want to be unique. In our database we'll also add `constraint exclusive` to `name` inside `Place` because these places are also all unique:

```
abstract type Place {
  required property name -> str {
      constraint exclusive;
  };
  property modern_name -> str;
  property important_places -> array<str>;
}
```

## Passing constraints with delegated

Now that our `Person` type has `constraint exclusive` for the property `name`, no type extending `Person` will be able to have the same name. That's fine for our game in this tutorial, because we already know all the character names in the book and won't be making any real `PC` types. But what if we later on wanted to make a `PC` named Jonathan Harker? Right now it wouldn't be allowed because we have an `NPC` with the same name, and `NPC` takes `name` from `Person`.

Fortunately there's an easy way to get around this: the keyword `delegated` in front of `constraint`. That "delegates" (passes on) the constraint to the subtypes, so the check for exclusivity will be done individually for `PC`, `NPC`, `Vampire`, and so on. So the type is exactly the same except for this keyword:

```
abstract type Place {
  required property name -> str {
      delegated constraint exclusive;
  };
  property modern_name -> str;
  property important_places -> array<str>;
}
```

With that you can have up to one Jonathan Harker the `PC`, the `NPC`, the `Vampire`, and anything else that extends `Place`.

## Using functions in queries

Let's also think about our game mechanics a bit. The book says that the doors inside the castle are too tough for Jonathan to open, but Dracula is strong enough to open them all. In a real game it will be more complicated but we can try something simple to mimic this:

- Doors have a strength, and people have strength as well. 
- If a person has greater strength than the door, then he or she can open it. 

So we'll create a type `Castle` and give it some doors. For now we only want to give it some "strength" numbers, so we'll just make it an `array<int16>`:

```
type Castle extending Place {
    property doors -> array<int16>;
}
```

Then we'll imagine that there are three main doors to enter and leave Castle Dracula, so we `INSERT` them as follows:

```
INSERT Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};
```

Then we will also add a `property strength -> int16;` to our `Person` type. It won't be required because we don't know the strength of everybody in the book...though later on we could make it required if the game needs it.

Now we'll give Jonathan a strength of 5. That's easy with `UPDATE` and `SET` like before:

```
UPDATE Person FILTER .name = 'Jonathan Harker'
  SET {
    strength := 5
};
```

Great. We know that Jonathan can't break out of the castle, but let's try to show it using a query. To do that, he needs to have a strength greater than that of a door. Or in other words, he needs a greater strength than the weakest door.

Fortunately, there is a function called `min()` that gives the minimum value of a set, so we can use that. If his strength is higher than the door with the smallest number, then he can escape. This query looks like it should work, but not quite:

```
WITH 
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  castle_doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
    SELECT jonathan_strength > min(castle_doors);
```

Here's the error:

```
error: operator '>' cannot be applied to operands of type 'std::int16' and 'array<std::int16>'
```

We can [look at the function signature](https://edgedb.com/docs/edgeql/funcops/set#function::std::min) to see the problem:

```
std::min(values: SET OF anytype) -> OPTIONAL anytype
```

The important part is `SET OF`: it needs a set, so something in curly brackets. We can't just put curly brackets around the array, because then it becomes a set of one item (one array). So `SELECT min({[5, 6]});` just returns `{[5, 6]}`, not `{5}`, because `{[5, 6]}` is the minimum value of the arrays we gave it...because we only gave it one array to look at.

That also means that `SELECT min({[5, 6], [2, 4]});` will give us the output `{[2, 4]}` (instead of 2). That's not what we want.

Instead, what we want to use is the [array_unpack()](https://edgedb.com/docs/edgeql/funcops/array#function::std::array_unpack) function which takes an array and unpacks it into a set. So we'll use that on `weakest_door`:

```
WITH 
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
    SELECT jonathan_strength > min(array_unpack(doors));
```

That gives us `{false}`. Perfect! Now we have shown that Jonathan can't open any doors. He will have to climb out the window to escape.

Along with `min()` there is of course `max()`. `len()` and `count()` are also useful: `len()` gives you the length of an object, and `count()` the number of them. Here is an example of `len()` to get the name length of all the `NPC`  types:

```
SELECT (NPC.name, 'Name length is: ' ++ <str>len(NPC.name));
```

Don't forget that we need to cast with `<str>` because `len()` returns an integer, and EdgeDB won't concatenate a string to an integer. This prints:

```
{
  ('The innkeeper', 'Name length is: 13'),
  ('Mina Murray', 'Name length is: 11'),
  ('Jonathan Harker', 'Name length is: 15'),
}
```

The other example is with `count()`, which also has a cast to a `<str>`:

```
SELECT 'There are ' ++ <str>(SELECT count(Place) - count(Castle)) ++ ' more places than castles';
```

It prints: `{'There are 6 more places than castles'}`.

In a few chapters we will learn how to create our own functions to make queries shorter.

## Using $ to set parameters

Imagine we need to look up `City` types all the time, with this sort of query:

```
SELECT City {
  name,
  population
  } FILTER .name ILIKE '%a%' AND len(.name) > 5 AND .population > 6000;
```

This works fine, returning one city: `{Object {name: 'Buda-Pesth', population: 402706}}`.

But this last line with all the filters can be a little annoying to change: there's a lot of moving about to delete and retype before we can hit enter again.

This could be a good time to add parameters to a query by using `$`. With that we can give them a name, and EdgeDB will ask us for every query what value to give it. Let's start with something very simple:

```
SELECT City {
  name
  } FILTER .name = 'London';
```

Now let's change 'London' to `$name`. Note: this won't work yet. Try to guess why!

```
SELECT City {
  name
  } FILTER .name = $name;
```

The problem is that `$name` could be anything, and EdgeDB doesn't know what type it's going to be. The error tells us too: `error: missing a type cast before the parameter`. So because it's a string, we'll cast with `<str>`:

```
SELECT City {
  name
  } FILTER .name = <str>$name;
```

When we do that, we get a prompt asking us to enter the value: `Parameter <str>$name:` Just type London, with no quotes because it already knows that it's a string. The result: `{Object {name: 'London'}}`

Now let's take that to make a much more complicated (and useful) query, using three parameters. We'll call them `$name`, `$population`, and `$length`. Don't forget to cast them all:

```
SELECT City {
  name,
  population,
  } FILTER
  .name ILIKE '%' ++ <str>$name ++ '%' 
  AND 
  .population > <int64>$population
  AND 
  <int64>len(.name) > <int64>$length;
```

Since there are three of them, EdgeDB will ask us to input three values. Here's one example of what it looks like:

```
Parameter <str>$name: u
Parameter <int64>$population: 2000
Parameter <int64>$length: 5
```

So that will give all `City` types with u in the name, population of more than 2000, and a name longer than 5 characters. The result:

```
{
  Object {name: 'Buda-Pesth', population: 402706},
  Object {name: 'Munich', population: 230023},
}
```

Parameters work just as well in inserts too. Here's a `Date` insert that prompts the user for the hour, minute, and second:

```
SELECT(INSERT Date {
 date := <str>$hour ++ <str>$minute ++ <str>$second
 }) {
 date,
 local_time,
 hour,
 awake
};
Parameter <str>$hour: 10
Parameter <str>$minute: 09
Parameter <str>$second: 09
```

Here's the output:

```
{
  default::Date {
    date: '100909',
    local_time: <cal::local_time>'10:09:09',
    hour: '10',
    awake: 'asleep',
  },
}
```
  
Note that the cast means you can just type 10, not '10'.

So what if you just want to have the *option* of a parameter? No problem, just put `OPTIONAL` before the type name in the cast (inside the `<>` brackets). So the insert above would look like this if you wanted everything optional:

```
SELECT(INSERT Date {
 date := <OPTIONAL str>$hour ++ <OPTIONAL str>$minute ++ <OPTIONAL str>$second
 }) {
 date,
 local_time,
 hour,
 awake
};
```

Of course, the `Date` type needs the proper formatting for the `date` property so this is a bad idea. But that's how you would do it.

The opposite of `OPTIONAL` is `REQUIRED`, but it's the default so you don't need to write it.

The `UPDATE` keyword that we learned last chapter can also take parameters, so that's three in total where you can use them: `SELECT`, `INSERT`, and `UPDATE`.

[Here is all our code so far up to Chapter 7.](code.md)

## Time to practice

<!-- quiz-start -->

1. How would you select each City and the length of its name?

2. How would you select each City and the length of `name` minus the length of `modern_name` if `modern_name` exists, and 0 if `modern_name` does not exist?

3. What if you wanted to write 'Modern name does not exist' instead of 0?

4. How would you insert an NPC with the name 'NPC number 8' if for example there are already seven other NPCs?

5. How would you select only the `Person` types that have the shortest names?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 8: [Workers in the city of Varna load fifty boxes into a ship. Dracula is inside one of them.](../chapter8/index.md)
