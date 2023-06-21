---
tags: Constraint Delegation, $ Parameters
---

# Chapter 7 - Jonathan finally "leaves" the castle

> Jonathan sneaks into Dracula's room during the day and sees him sleeping inside a coffin. Now Jonathan knows that Count Dracula is a vampire.
>
> A few days later Count Dracula says that he will leave tomorrow. Jonathan thinks this is a chance, and asks to leave now. Dracula says, "Fine, if you wish..." and opens the door, but there are a lot of wolves outside, howling and making loud sounds. Dracula says, "You are free to leave! Goodbye!" But Jonathan knows that the wolves will kill him if he steps outside. Jonathan also knows that Dracula called the wolves, and asks him to please close the door. Dracula smiles and closes the door...he knows that Jonathan is trapped.
>
> Later, Jonathan hears Dracula tell the vampire women that he will leave the castle tomorrow and that they can have Jonathan at that time. Dracula's friends take him away inside a coffin the next day, and Jonathan is alone...and soon it will be night. All the doors are locked. Jonathan decides to climb out the window, because it is better to die by falling than to be alone with the vampire women.
>He writes "Good-bye, all! Mina!" in his journal and begins to climb the wall.

## More constraints

While Jonathan climbs the wall, we can continue to work on our database schema. In our book, no character has the same name so there should only be one Mina Murray, one Count Dracula, and so on. This is a good time to put a {ref}`constraint <docs:ref_datamodel_constraints>` on `name` in the `Person` type to make sure that we don't have duplicate inserts. A `constraint` is a limitation, which we saw already in `age` for humans that can only go up to 120: `constraint max_value(120);`.

We can give `name` a constraint too called `constraint exclusive` which prevents two objects of the same type from having the same name. Like the other constraint we added, you put it `constraint` in a block after the property, like this:

```sdl
abstract type Person {
  required name: str { ## Add a block
      constraint exclusive;       ## and the constraint
  }
  multi places_visited: Place;
  lover: Person;
}
```

Now we know that there will only be one `Jonathan Harker`, one `Mina Murray`, and so on. In real life this is often useful for email addresses, User IDs, and other properties that you always want to be unique. In our database we'll also add `constraint exclusive` to `name` inside `Place` because these places are also all unique:

```sdl
abstract type Place {
  required name: str {
      constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

Now let's do a migration. At this point, when you type `migration create` the database will apply the constraint to the existing objects. If one of them violates the constraint then the migration will fail until you change the objects to match the constraint. For example, if we had two `MinorVampire` objects with the same name, it would give this output:

```
Detail: value of property 'name' of object type 'default::MinorVampire' violates exclusivity constraint
```

## Passing constraints with delegated

Now that our `Person` type has `constraint exclusive` for the property `name`, no type extending `Person` will be able to have the same name. That's fine for our game in this tutorial, because we already know all the character names in the book and won't be making any real `PC` type objects. But what if we later on wanted to make a `PC` named Jonathan Harker? Right now it wouldn't be allowed because we have an `NPC` with the same name, and `NPC` takes `name` from `Person`.

Fortunately there's an easy way to get around this: the keyword `delegated` in front of `constraint`. That "delegates" (passes on) the constraint to the subtypes, so the check for exclusivity will be done individually for `PC`, `NPC`, `Vampire`, and so on. So the type is exactly the same except for this keyword:

```sdl
abstract type Person {
  required name: str {
    delegated constraint exclusive;
  }
  multi places_visited: Place;
  lover: Person;
  strength: int16;
}
```

With that you can have up to one Jonathan Harker the `PC`, the `NPC`, the `Vampire`, and anything else that extends `Person`.

Also, the `delegated constraint` applies to `Place`, since for example `Country` can have the same name as `City`. So let's update the `name` property for the `Place` type:

```sdl
abstract type Place {
  required name: str {
      delegated constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

## Using functions in queries

Let's also think about our game mechanics a bit. The book says that the doors inside the castle are too tough for Jonathan to open, but Dracula is strong enough to open them all. In a real game it will be more complicated but we can try something simple to mimic this:

- Doors have a strength, and people have strength as well.
- A `Person` with greater strength than the door will be able to open it.

So we will change our `Castle` type to give it some doors. For now we only want to give it some "strength" numbers, so we'll just make it an `array<int16>`:

```sdl
type Castle extending Place {
    doors: array<int16>;
}
```

Then we will also add a `strength: int16;` to our `Person` type. It won't be required because we don't know the strength of everybody in the book. Plus, if we made it a `required` property, we would have to choose a default strength for every `Person` object that we already have.

Now it's time to do an insert. We'll imagine that there are three main doors to enter and leave Castle Dracula. First let's update the schema with `edgedb migration create` and `edgedb migrate` as usual.

Now we have to add the doors to Castla Dracula, so let's update it:

```edgeql
update Castle filter .name = 'Castle Dracula'
  set {
    doors := [6, 9, 10]
  };
```

Now we'll give Jonathan a strength of 5.  And now we can update Jonathan with `update` and `set` like before:

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  strength := 5
};
```

Great. We can see that Jonathan doesn't have enough strength to break out of the castle, but let's try to show it using a query. To do that, he needs to have a strength greater than that of any a door. Or in other words, he needs a greater strength than the weakest door.

Fortunately, there is a function called `min()` that gives the minimum value of a set, so we can use that. If his strength is higher than the door with the smallest number, then he can escape. This query looks like it should work, but not quite:

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  castle_doors := (select Castle filter .name = 'Castle Dracula').doors,
select jonathan_strength > min(castle_doors);
```

Here's the error:

```
error: InvalidTypeError: operator '>' cannot be applied to 
operands of type 'std::int16' and 'array<std::int16>'
```

We can {eql:func}`look at the function signature <docs:std::min>` to see the problem:

```
std::min(values: set of anytype) -> optional anytype
```

The important part is `set of`: it needs a set, so something in curly brackets. We can't just put curly brackets around the array, because then it becomes a set of one item (one array). So `select min({[5, 6]});` just returns `{[5, 6]}`, not `{5}`, because `{[5, 6]}` is the minimum value of the arrays we gave it...because we only gave it one array to look at.

That also means that `select min({[5, 6], [2, 4]});` will give us the output `{[2, 4]}` (instead of 2). That's not what we want.

Instead, what we want to use is the {eql:func}` ``array_unpack()`` <docs:std::array_unpack>` function which takes an array and unpacks it into a set. So we'll use that on `castle_doors`:

```edgeql
with
  jonathan := (select Person filter .name = 'Jonathan Harker'),
  castle := (select Castle filter .name = 'Castle Dracula'),
  select jonathan.strength > min(array_unpack(castle.doors));
```

That gives us `{false}`. Perfect! Now we have shown that Jonathan can't open any doors. He will have to climb out the window to escape.

Along with `min()` there is of course `max()`. `len()` and `count()` are also useful: `len()` gives you the length of an object, and `count()` the number of them. Here is an example of `len()` to get the name length of all the `NPC` type objects:

```edgeql
select (NPC.name, 'Name length is: ' ++ <str>len(NPC.name));
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

```edgeql
select 'There are ' ++ <str>(select count(Place) 
      - count(Castle)) ++ ' more places than castles';
```

It prints: `{'There are 8 more places than castles'}`. (The number 8 might be a bit different from yours if you have been experimenting with inserting `Place` objects.)

In a few chapters we will learn how to create our own functions to make queries like these shorter. Once you learn to make your own functions you will be able to write something short like `select can_escape('Jonathan Harker', 'Castle Dracula');` and the function will do the rest! But in the meantime let's move on to a similar subject: setting parameters in queries.

## Using $ to set parameters

Imagine we need to look up `City` type objects all the time, with this sort of query:

```edgeql
select City {
  name,
  modern_name
} filter .name ilike '%i%' and exists (.modern_name);
```

This works fine, returning one city:

```
{default::City {name: 'Bistritz', modern_name: 'Bistrița'}}
```

But this last line with all the filters can be a little annoying to change: there's a lot of moving about to delete and retype before we can hit enter again. Or we might be using EdgeDB through one of its [client libraries](https://www.edgedb.com/docs/clients/index) and would like to pass in parameters instead of rewriting the query every time.

This could be a good time to add parameters to a query by using `$`. When EdgeDB sees the `$` it knows that this must be replaced with a value, and on the REPL it will ask us what value to give it. Let's start with something very simple:

```edgeql
select City {
  name
} filter .name = 'London';
```

Now let's change 'London' to `$name`. Note: this won't work yet. Try to guess why!

```edgeql
select City {
  name
} filter .name = $name;
```

The problem is that `$name` could be anything, and EdgeDB doesn't know what type it's going to be. The error gives us a hint for what to do:

```
error: QueryError: missing a type cast before the parameter
  ┌─ <query>:3:18
  │
3 │ } filter .name = $name;
  │                  ^^^^^ error
```

In this case we will enter a `str`, and can use `<str>` to let EdgeDB know ahead of time that this is the type to expect:

```edgeql
select City {
  name
} filter .name = <str>$name;
```

When we do that, we get a prompt asking us to enter the value:

```
Parameter <str>$name:
```

Just typing London and hitting enter will lead to this expected result:

```
{default::City {name: 'London'}}
```

Note that on the REPL it knows to expect a string so you don't need to type `'London'`. Give `'London'` a try though! The query works, but returns an empty set: `{}`. That's because it's looking for a `City` object where the name is `'London'`, not `London`.

Now let's take that to make a much more complicated (and useful) query, using two parameters. We'll call them `$name` and `$has_modern_name`. Don't forget to cast them all:

```edgeql
select City {
  name,
  modern_name
} filter
    .name ilike '%' ++ <str>$name ++ '%'
  and
    exists (.modern_name) = <bool>$has_modern_name;
```

Since there are two of them, EdgeDB will ask us to input two values. Here's one example of what it looks like:

```
Parameter <str>$name: b
Parameter <bool>$has_modern_name: true
```

So that will give all `City` type objects with "b" in the name and that have a different modern name. In our case, objects with the `modern_name` property have it because their modern name is different from the name in the book. The result:

```
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

Parameters work just as well in inserts too. Here's a `Time` insert that prompts the user for the hour, minute, and second:

```
with time := (
   insert Time {
     clock := <str>$hour ++ <str>$minute ++ <str>$second
   }
 ),
 select time {
 clock,
 clock_time,
 hour,
 sleep_state
 };
Parameter <str>$hour: 10
Parameter <str>$minute: 09
Parameter <str>$second: 09
```

Here's the output:

```
{
  default::Time {
    clock: '100909',
    clock_time: <cal::local_time>'10:09:09',
    hour: '10',
    sleep_state: 'asleep',
  },
}
```

Note that the cast means you can just type 10, not '10'.

## Optional parameters

So what if you just want to have the _option_ of a parameter? No problem, just put `optional` before the type name inside the cast (inside the `<>` brackets). We could use this to change the query on `City` object names above to allow a second filter for letters in the name:

```edgeql
select City {
  name,
  modern_name
} filter
    .name ilike '%' ++ <str>$input1 ++ '%'
  and
  .name ilike '%' ++ <optional str>$input2 ++ '%';
```

In this case you could search for cities containing both `B` and `z` (which would return `Bistritz` but not `Buda-Pesth`), or just search for cities containing `B` and not enter anything for the second input.

The opposite of `optional` is `required`, but `required` is the default so you don't need to write it.

The `update` keyword that we learned last chapter can also take parameters, so that's four in total where you can use them: `select`, `insert`, `update`, and `delete`.

[Here is all our code so far up to Chapter 7.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you select each City and the length of its name?

2. How would you select each City and the length of `name` minus the length of `modern_name` if `modern_name` exists, and 0 if `modern_name` does not exist?

3. What if you wanted to write 'Modern name does not exist' instead of 0?

4. How would you insert an NPC with the name 'NPC number 8' if for example there are already seven other NPCs?

5. How would you select only the `Person` type objects that have the shortest names?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Workers in the city of Varna load boxes into a ship. Dracula is inside one of them..._
