---
tags: Constraint Delegation, $ Parameters
---

# Chapter 7 - Jonathan finally "leaves" the castle

> Jonathan sneaks into Dracula's room during the day and sees him sleeping inside a coffin. Now Jonathan knows that Count Dracula is a vampire.
>
> A few days later Count Dracula comes with some news. He tells Jonathan that he will leave the castle tomorrow, and that Jonathan's stay at the castle has also come to an end. Jonathan thinks this is a chance, and asks to leave now instead of tomorrow.
>
> Dracula says, "Fine, if you wish..." and opens the door, but there are a lot of wolves outside, howling and getting ready to attack. Dracula smiles and begins to open the door as he says, "You are free to leave! Goodbye!" Jonathan knows that Dracula called the wolves, and panics. "Shut the door! I shall wait till morning!" says Jonathan. Dracula laughs, slams the door shut and walks away.
>
> Later, Jonathan hears Dracula tell the vampire women they can have Jonathan once he is alone in the castle. Some workers come to take Dracula away inside a coffin the next day, and Jonathan is alone...and soon it will be night. All the doors are locked. Jonathan has no choice and decides to climb out the window. It is better to die by falling than to be alone with the vampire women.
>
> After writing "Good-bye, all! Mina!" in his journal, Jonathan begins to climb the wall.

## More constraints and simple ordering

While Jonathan climbs the wall, we can continue to work on our database schema. Let's give it some more constraints so that we are sure what data is acceptable and what is not.

No character in our book has the same name, so there should only be one Mina Murray, one Count Dracula, and so on. No `PC` object should have the same name either: imagine that you created a `PC` to play the game but the next day someone else shows up with the same name as you! Even worse, any `update` done to a `PC filter .name = your_name` might end up updating both characters at the same time.

To avoid this, we can put a {ref}`constraint <docs:ref_datamodel_constraints>` on `name` in the `Person` type to make sure that we don't have duplicate inserts. A `constraint` is a limitation, which we saw already in `age` for humans that can only go up to 120:

```constraint max_value(120);```

We can give `name` a constraint too called `constraint exclusive` which prevents two objects of the same type from having the same value - in this case, a name. Like the other constraint we added, you put `constraint` in a block after the property, like this:

```sdl
abstract type Person {
  required name: str {      # Add a block
      constraint exclusive; # and the constraint
  }
  multi places_visited: Place;
  lover: Person;
  is_single := not exists .lover;
}
```

With this constraint added, we now know that there will only be one `Jonathan Harker`, one `Mina Murray`, and so on. In real life this is often useful for email addresses, User IDs, and other properties that you always want to be unique. In our database we'll also add `constraint exclusive` to `name` inside `Place` because these places are also all unique:

```sdl
abstract type Place {
  required name: str {
      constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

We are going to do a migration now, but first let's insert an object that will violate the `exclusive` constraint. Remember the innkeeper from the city of Bistritz? Let's add him again:

```edgeql
insert NPC { name := 'The innkeeper' };
```

Great! Now our migration is going to fail. However, `edgedb migration create` will work, because this simply creates the commands to carry out the migration. After that comes `migration create`, which is when the database will apply the constraint to the existing objects. Fortunately, the output will tell us what has gone wrong:

```
edgedb error: ConstraintViolationError: name violates exclusivity constraint
Detail: property 'name' of object type 'default::NPC' violates exclusivity constraint
edgedb error: error in one of the migrations
```

"Property 'name' of object type 'default::NPC' violates exclusivity constraint" is a pretty clear error message.

```edgeql
select NPC { name };
```

The output shows us that there are two `NPC` objects called 'The innkeeper', which is not okay in our new schema.

```
{
  default::NPC {name: 'The innkeeper'},
  default::NPC {name: 'Mina Murray'},
  default::NPC {name: 'Jonathan Harker'},
  default::NPC {name: 'The innkeeper'},
}
```

We are going to have to delete one, but let's order those results first. After all, there could have been 10 or 20 or more objects and trying to find a duplicate name would have been pretty tough.

Ordering is pretty easy: just add `order by` and the property to order by: 

```edgeql
select NPC { name } order by .name;
```

Now the output shows 'The innkeeper' right next to the other object of the same name.

```
{
  default::NPC {name: 'Jonathan Harker'},
  default::NPC {name: 'Mina Murray'},
  default::NPC {name: 'The innkeeper'},
  default::NPC {name: 'The innkeeper'},
}
```

We will learn more about ordering in Chapter 10. But in the meantime, let's get back to our duplicate objects so we can delete one. They are identical in every way except their `id`, so let's find out what they are:

```
select NPC { id } filter .name = 'The innkeeper';
```

Your `id` values will be different, but the output will look like this:

```
{
  default::NPC {id: ebe395c4-19cc-11ee-bae7-f7a7bff901b9},
  default::NPC {id: dbb3bb4c-19e6-11ee-9981-03a7bead0c6b},
}
```

And now we'll just pick one id, put it inside a `str` and cast it to a `uuid` as a filter to delete the one `NPC` object. 

```edgedb
delete NPC filter .id = <uuid>'ebe395c4-19cc-11ee-bae7-f7a7bff901b9';
```

With the offending object gone, the `edgedb migrate` command now works!

## Passing constraints with delegated

Now that our `Person` type has `constraint exclusive` for the property `name`, no type extending `Person` will be able to have the same name. That's fine for our game in this tutorial, because we already know all the character names in the book and won't be making many real `PC` type objects. But what if we later on wanted to make a `PC` named Jonathan Harker? Right now it wouldn't be allowed because we have an `NPC` with the same name, and `NPC` takes `name` from `Person`.

Fortunately there's an easy way to get around this: by putting the keyword `delegated` in front of `constraint`. That "delegates" (passes on) the constraint to the subtypes, so that the check for exclusivity will be done individually for `PC`, `NPC`, `Vampire`, and so on. That makes the `Person` type exactly the same except for the `delegated` keyword:

```sdl
abstract type Person {
  required name: str {
    delegated constraint exclusive;
  }
  multi places_visited: Place;
  lover: Person;
  is_single := not exists .lover;
}
```

With that you can have up to one Jonathan Harker the `PC`, the `NPC`, the `Vampire`, and anything else that extends `Person`.

The `delegated constraint` should also apply to `Place` since a `Country` can have the same name as `City`, and so on for any other types that will extend `Place`. So let's update the constraint on the `name` property for the `Place` type to add `delegated` there too.

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

Let's also think about our game mechanics a bit. The book says that the doors inside the castle are too tough for Jonathan to open, but Dracula is strong enough to open them all. In a real game it would be more complicated but we can try something simple to mimic this:

- Doors have a strength, and people have strength as well.
- A `Person` with greater strength than the door will be able to open it.

So we will change our `Castle` type to give it some doors. For now we only want to give it some "strength" numbers, so we'll just make it an `array<int16>`:

```sdl
type Castle extending Place {
    doors: array<int16>;
}
```

Then we will also add a `strength: int16;` to our `Person` type. This property won't be `required` because we don't know the strength of everybody in the book. Plus, if we made it a `required` property, we would have to choose a default strength for every `Person` object that we already have.

Now it's time to do an insert. We'll imagine that there are three main doors to enter and leave Castle Dracula. First let's update the schema with `edgedb migration create` and `edgedb migrate` as usual.

Now we have to add the doors to Castla Dracula, so let's update it:

```edgeql
update Castle filter .name = 'Castle Dracula'
  set {
    doors := [6, 9, 10]
  };
```

Now we'll give Jonathan a strength of 5. That's another easy `update`:

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  strength := 5
};
```

We can see that Jonathan doesn't have enough strength to break out of the castle, but let's try to show it using a query. To do that, he needs to have a strength greater than that of any a door. Or in other words, he needs a greater strength than the weakest door.

Fortunately, there is a function called `min()` that gives the minimum value of a set, so we can use that. If his strength is higher than the door with the smallest number, then he can escape. This query looks like it should work, but not quite:

```edgeql
with
  jonathan := (select Person filter .name = 'Jonathan Harker'),
  castle   := (select Castle filter .name = 'Castle Dracula'),
select jonathan.strength > min(castle.doors);
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
  castle   := (select Castle filter .name = 'Castle Dracula'),
  select jonathan.strength > min(array_unpack(castle.doors));
```

That gives us `{false}`. Perfect! Now we have shown that Jonathan can't open any doors. He will have to climb out the window to escape.

Unsurprisingly, along with `min()` there is also a function called `max()`. `len()` and `count()` are also useful: `len()` gives you the length of an object, and `count()` the number of them. Here is an example of `len()` to get the name length of all the `NPC` type objects:

```edgeql
select ('Length of "' ++ NPC.name ++ '" is: ' ++ <str>len(NPC.name));
```

Don't forget that we need to cast with `<str>` because `len()` returns an integer, and EdgeDB won't concatenate a string to an integer. Here is the result:

```
{
  'Length of "Mina Murray" is: 11',
  'Length of "Jonathan Harker" is: 15',
  'Length of "The innkeeper" is: 13',
}
```

This next example uses `count()`, which also uses a cast to a `<str>`:

```edgeql
select 'There are ' ++ <str>(select count(Place) 
      - count(Castle)) ++ ' more places than castles';
```

It prints: `{'There are 8 more places than castles'}`. Or your query might return a different number if you have been experimenting with inserting `Place` objects.

In Chapter 11 we will learn how to write our own functions to make queries like these shorter. Once you learn to make your own functions you will be able to write something short like `select can_escape('Jonathan Harker', 'Castle Dracula');` and the function will do the rest! But in the meantime let's move on to a similar subject: setting parameters in queries.

## Using $ to set parameters

Imagine we need to look up `City` type objects all the time, with this sort of query:

```edgeql
select Place {
  name,
  modern_name
} filter .name ilike '%i%' and exists .modern_name;
```

This works fine, returning one city:

```
{default::City {name: 'Bistritz', modern_name: 'Bistrița'}}
```

But this last line with all the filters can be a little annoying to change: there's a lot of moving about to delete and retype before we can hit enter again. Or we might be using EdgeDB through one of its [client libraries](https://www.edgedb.com/docs/clients/index) for languages like TypeScript, Python and Rust, and would like to pass in parameters instead of rewriting the query every time.

This could be a good time to add parameters to a query by using `$`. When EdgeDB sees the `$` it knows that this must be replaced with a value, and in the REPL it will ask us what value to give it. Let's start with something very simple:

```edgeql
select Place {
  name
} filter .name ilike '%ondon%';
```

No surprise here: this will return the `City` object with the name `London`.

Now let's change 'London' to `$name`. Note: this won't work yet. Try to guess why!

```edgeql
select Place {
  name
} filter .name ilike $name;
```

The problem is that `$name` could be anything, and EdgeDB doesn't know what type it's going to be. The error gives us a hint for what to do:

```
error: QueryError: missing a type cast before the parameter
  ┌─ <query>:3:18
  │
3 │ } filter .name ilike $name;
  │                      ^^^^^ error
```

In this case we want to enter a `str`, so we can use `<str>` to let EdgeDB know ahead of time that this is the type to expect.

```edgeql
select Place {
  name
} filter .name ilike <str>$name;
```

When we do that we get a prompt asking us to enter the value:

```
Parameter <str>$name:
```

And now, just typing `%ondon%` or `London` and hitting enter will lead to this expected result:

```
{default::City {name: 'London'}}
```

Here are two points to keep in mind before we continue:

* The REPL now knows to expect a string so you don't need to surround it with quotes. Give `'London'` a try though and see what happens! The query works, but returns an empty set: `{}`. That's because it's looking for a `City` object where the name is `'London'`, not `London`.
* The `<>` cast notation in EdgeDB actually has two uses: casting and type specification (letting the compiler know which type to expect). In this case, it is being used for type specification. That means that the compiler is not using `<str>` to cast input into a `str`, but simply to know to expect a `str` - and to reject input that is of a different type. The REPL is smart enough to not allow us to give it improper input when it expects a `str`, but if you are using a client library then there is no REPL to check a query before you send it to EdgeDB. So make sure that you are sending a string when it expects a `str`!

Now let's use what we know to make a more useful query, using two parameters. We'll call them `$name` and `$name_has_changed`. Don't forget to use the cast notation for both:

```edgeql
select Place {
  name,
  modern_name
} filter
    .name ilike '%' ++ <str>$name ++ '%'
  and
    exists .modern_name = <bool>$name_has_changed;
```

Since there are two of them, EdgeDB will ask us to input two values. Here's one example of what it looks like:

```
Parameter <str>$name: b
Parameter <bool>$name_has_changed: true
```

So that will give all `Place` type objects with "b" in the name and which have a different name today than their name in the book. In our case, objects with the `modern_name` property have it because their modern name is different from the name in the book. The result:

```
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

Parameters work just as well in inserts too. Here's an update for our global `Time` object that prompts the user for the hour, minute, and second:

```
with new_time := <str>$hour ++ ':' ++ <str>$minute ++ ':' ++ <str>$second,
 current_time := (update Time set {
 clock := new_time
 })
 select current_time {*};
Parameter <str>$hour: 20
Parameter <str>$minute: 19
Parameter <str>$second: 00
```

And the output:

```
{
  default::Time {
    id: eae7edc8-19cc-11ee-bae7-e3434cce8ad7,
    clock: '20:19:00',
    clock_time: <cal::local_time>'20:19:00',
    hour: '20',
    vampires_are: Awake,
  },
}
```

After doing this query, our `global time` will be updated as well:

```edgeql
select global time {*};
```

The output will be the same as the `Time` object directly above.

## Optional parameters and the coalescing operator

There is also a way to do queries that just give the _option_ of a parameter. To do this, just put `optional` before the type name inside the cast (inside the `<>` brackets). We could use this to change the query on `Place` object names above to allow a second filter for letters in the name.

With an optional parameter you could search for places that:

* contain both `B` and `z` (which would return `Bistritz` but not `Buda-Pesth`), or
* contain `B`, and not provide anything for the second input. In this case the query would return both `Bistritz` and `Buda-Pesth`.

The opposite of `optional` is `required`, but `required` is the default so you don't need to write it.

Putting all this together ends up with a query like the following. Note that we want to check to see if the optional parameter `exists`, and to only filter on the required parameter if it doesn't.

```edgeql
with
  f1 := <str>$filter_1,
  f2 := <optional str>$filter_2,
 select Place {
   name,
   modern_name
 } filter 
   .name ilike '%' ++ f1 ++ '%' and .name ilike '%' ++ f2 ++ '%' 
     if exists f2 else 
   .name ilike '%' ++ f1 ++ '%';
```

Here are two sample outputs for this query from the REPL:

```
Parameter <str>$filter_1: B
Parameter <str>$filter_2 (Ctrl+D for empty set `{}`): z
{default::City {name: 'Bistritz', modern_name: 'Bistrița'}}
```

And:

```
Parameter <str>$filter_1: B
Parameter <str>$filter_2 (Ctrl+D for empty set `{}`):
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

The second parameter which asks us if we want to enter an empty string or an empty set is interesting, and has to do with some concepts called "Cartesian multiplication" and the "coalescing operator". But those subjects are too large to fit into the end of this chapter, so we'll have to wait until Chapter 11 to learn them.

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
