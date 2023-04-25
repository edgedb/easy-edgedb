---
tags: Tuples, Computed Properties, Math
---

# Chapter 10 - Terrible events in Whitby

> Mina and Lucy are enjoying their time in Whitby. One night there is a huge storm and a ship arrives in the fog - it's the Demeter, carrying Dracula. Lucy later begins to sleepwalk at night and looks very pale, and always says strange things. Mina tries to stop her, but sometimes Lucy gets outside. One night Lucy watches the sun go down and says: "His red eyes again! They are just the same." Mina is worried and asks Dr. Seward for help. Dr. Seward does an examination on Lucy. She is pale and weak, but he doesn't know why. Dr. Seward decides to call his old teacher Abraham Van Helsing, who comes from the Netherlands to help. Van Helsing examines Lucy and looks shocked. Then he turns to the others and says, "Listen. We can help this girl, but you are going to find the methods very strange. You are going to have to trust me..."

The city of Whitby is in the northeast of England. Right now our `City` type just extends `Place`, which only gives us the properties `name`, `modern_name` and `important_places`. This could be a good time to give it a `property population` which can help us draw the cities in our game. It will be an `int64` to give us the size we need:

```sdl
type City extending Place {
  property population -> int64;
}
```

By the way, here are the approximate populations for our three cities at the time of the book. They are much smaller back in 1887:

- Buda-Pesth (Budapest): 402706
- London: 3500000
- Munich: 230023
- Whitby: 14400
- Bistritz (Bistrița): 9100

Inserting Whitby is easy enough:

```edgeql
insert City {
  name := 'Whitby',
  population := 14400
};
```

But for the rest of them it would be nice to update everything at the same time.

## Working with tuples and arrays

If we have all the city data together, we can do a single insert with a `for` and `union` loop again. Let's imagine that we have some data inside a tuple, which seems similar to an array but is quite different. One big difference is that a tuple can hold different types, so this is okay:

`('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)`

In this case, the type is called a `tuple<str, int64>`.

Before we start using these tuples, let's make sure that we understand the difference between the two. To start, let's look at slicing arrays and strings in a bit more detail.

You'll remember that we use square brackets to access part of an array or a string. So `select ['Mina Murray', 'Lucy Westenra'][1];` will give the output `{'Lucy Westenra'}` (that's index number 1).

You'll also remember that we can separate the starting and ending index with a colon, like in this example:

```edgeql
select NPC.name[0:10];
```

This prints the first ten letters of every NPC's name:

```
{
  'The innkee',
  'Mina Murra',
  'Jonathan H',
  'Lucy Weste',
  'John Sewar',
  'Quincey Mo',
  'Arthur Hol',
  'Renfield',
}
```

But the same can be done with a negative number if you want to start from the index at the end. For example:

```edgeql
select NPC.name[2:-2];
```

This prints from index 2 up to 2 indexes away from the end (it'll cut off the first two letters on each side). Here's the output:

```
{
  'e innkeep',
  'na Murr',
  'nathan Hark',
  'cy Westen',
  'hn Sewa',
  'incey Morr',
  'thur Holmwo',
  'nfie',
}
```

Tuples are quite different: they behave more like object types with properties that have numbers instead of names. This is why tuples can hold different types together: `string`s with `array`s, `int64`s with `float32`s, anything.

So this is completely fine:

```edgeql
select {('Bistritz', 9100, cal::to_local_date(1887, 5, 6)), ('Munich', 230023, cal::to_local_date(1887, 5, 8))};
```

The output is:

```
{
  ('Bistritz', 9100, <cal::local_date>'1887-05-06'),
  ('Munich', 230023, <cal::local_date>'1887-05-08'),
}
```

But now that the type is set (this one is type `tuple<str, int64, cal::local_date>`) you can't mix it up with other tuple types. So this is not allowed:

```edgeql
select {(1, 2, 3), (4, 5, '6')};
```

EdgeDB will give an error because it won't try to work with tuples that are of different types. It complains:

```
ERROR: QueryError: operator 'union' cannot be applied to operands of type 'tuple<std::int64, std::int64, std::int64>' and 'tuple<std::int64, std::int64, std::str>'
  Hint: Consider using an explicit type cast or a conversion function.
```

In the above example we could easily just cast the last string into an integer and EdgeDB will be happy again: `select {(1, 2, 3), (4, 5, <int64>'6')};`.

To access the fields of a tuple you still start from the number 0, but you write the numbers after a `.` instead of inside a `[]`. Now that we know all this, we can update all our cities at the same time. It looks like this:

```edgeql
for data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
union (
  update City filter .name = data.0
  set {
    population := data.1
  }
);
```

So it sends each tuple into the `for` loop, filters by the string (which is `data.0`) and then updates with the population (which is `data.1`).

Let's finish this section with a final note about casting. We know that we can cast into any scalar type, and this works for tuples of scalar types too. It uses the same format with `<>` except that you put it inside of `<tuple>`, like this:

```edgeql
with london := ('London', 3500000),
select <tuple<json, int32>>london;
```

That gives us this output:

```
{("\"London\"", 3500000)}
```

Here's another example if we need to do some math with floats on London's population:

```edgeql
with london := <tuple<json, float64>>('London', 3500000),
  select (london.0, london.1 / 23);
```

The output is `{("\"London\"", 152173.91304347827)}`.

## Ordering results and using math

Now that we have some numbers, we can start playing around with ordering and math. Ordering is quite simple: type `order by` and then indicate the property/link you want to order by. Here we order them by population:

```edgeql
select City {
  name,
  population
} order by .population desc;
```

This returns:

```
{
  default::City {name: 'London', population: 3500000},
  default::City {name: 'Buda-Pesth', population: 402706},
  default::City {name: 'Munich', population: 230023},
  default::City {name: 'Whitby', population: 14400},
  default::City {name: 'Bistritz', population: 9100},
}
```

What's `desc`? It means descending, so largest first and then going down. If we didn't write `desc` then it would have assumed that we wanted to sort ascending. You can also write `asc` (to make it clear to somebody reading the code for example), but you don't need to.

For some actual math, you can check out the functions in `std` {eql:func}`here <docs:std::sum>` as well as the `math` module {ref}`here <docs:ref_std_math>`. Instead of looking at each one, let's do a single big query to show some of them all together. To make the output nice, we will write it together with strings explaining the results and then cast them all to `<str>` so we can join them together using `++`.

```edgeql
with cities := City.population
select (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>math::mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>math::stddev(cities)
);
```

This used quite a few functions:

- `count()` to count the number of items,
- `all()` to return `{true}` if all items match and `{false}` otherwise,
- `sum()` to add them all together,
- `max()` to give the highest value,
- `min()` to give the lowest one,
- `math::mean()` to give the average,
- `any()` to return `{true}` if any item matches and `{false}` otherwise, and
- `math::stddev()` for the standard deviation.

The output also makes it clear how they work:

```
{
  (
    'Number of cities: 5',
    'All cities have more than 50,000 people: false',
    'Total population: 4156229',
    'Smallest and largest population: 9100, 3500000',
    'Average population: 831245.8',
    'At least one city has more than 5 million people: false',
    'Standard deviation: 1500876.8248',
  ),
}
```

`any()`, `all()` and `count()` are particularly useful in operations to give you an idea of your data.

## Importing modules using the `with` keyword

You can use the `with` keyword to import modules too. In the example above we used two functions from EdgeDB's `math` module: `math::mean()` and `math::stddev()`. Just writing `mean()` and `stddev()` would produce this error:

```
ERROR: InvalidReferenceError: function 'default::mean' does not exist
```

If you don't want to write the module name every time you can just import the module after `with`. Let's slip that into the query we just used. See if you can see what's changed:

```edgeql
with cities := City.population,
  module math
select (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>stddev(cities)
);
```

The output is the same, but we added an import of the `math` module, letting us just write `mean()` and `stddev()`.

You can also use `AS` to rename a module (well, to _alias_ a module) in the same way that you can rename a type. So this will work too:

```edgeql
with M as module math,
select M::mean(City.population);
```

That gives us the mean: `{831245.8}`.

## Some more computed properties for names

We saw in this chapter that Dr. Seward asked his old teacher Dr. Van Helsing to come and help Lucy. Here is how Dr. Van Helsing began his letter to say that he was coming:

```
Letter, Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc., to Dr. Seward.

“2 September.

“My good Friend,—
“When I have received your letter I am already coming to you.
```

The `Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc.` part is interesting. This might be a good time to think about more about the `name` property inside our `Person` type. Right now our `name` property is just a single string, but we have people with different types of names, in this order:

Title | First name | Last name | Degree

So there is 'Count Dracula' (title and name), 'Dr. Seward' (title and name), 'Dr. Abraham Van Helsing, M.D, Ph. D. Lit.' (title + first name + last name + degrees), and so on.

That would lead us to think that we should have titles like `first_name`, `last_name`, and `title` and then join them together using a computed property. But then again, not every character has these exact four parts to their name. Some others that don't are 'Woman 1' and 'The Innkeeper', and our game would certainly have a lot more of these. So it's probably not a good idea to get rid of `name` or always build names from separate parts. But in our game we might have characters writing letters or talking to each other, and they will have to use things like titles and degrees.

We could try a middle of the road approach instead. We'll keep `name`, and add some properties to `Person`:

```sdl
property title -> str;
property degrees -> str;
property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
```

We could try to do something fancier with `degrees` by making it an `array<str>` for each degree, but our game probably doesn't need that much precision. We are just using this for our conversation engine.

Now it's time to insert Van Helsing:

```edgeql
insert NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```

Now we can make use of these properties to liven up our conversation engine in the game. For example:

```edgeql
with helsing := (select NPC filter .name ilike '%helsing%')
select (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ ',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

By the way, the `\n` inside the string creates a new line, while `\t` moves it one tab to the right.

This gives us:

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.,
        I am sorry to say that I bring bad news about Lucy.',
  ),
}
```

In a standard database with users it's much simpler: get users to enter their first names, last names etc. and make each one a property.

## Other escape characters and raw strings

Besides `\n` and `\t` there are quite a few other escape characters - you can see the complete list {ref}`here <docs:ref_eql_lexical_str_escapes>`. Some are rare but hexadecimal with `\x` is a good example of one that might be useful.

If you want to ignore escape characters, put an `r` in front of the quote. Let's try it with the example above. Only the last part has an `r`:

```edgeql
with helsing := (select NPC filter .name ilike '%helsing%')
select (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ r',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

Now we get:

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.,\\n\\tI am sorry to say that I bring bad news about Lucy.',
  ),
}
```

Finally, there is a raw string literal that uses `$$` on each side. Anything inside this will ignore any and all quotation marks, so you won't have to worry about the string ending in the middle. Here's one example with a bunch of single and double quotes inside:

```edgeql
select $$ 
"Dr. Van Helsing would like to tell "them" 
about "vampires" and how to "kill" them, 
but he'd sound crazy."
$$;
```

Without the `$$` it will look like four separate strings with three unknown keywords between them, and will generate an error.

## All the scalar types

You now have an understanding of all the EdgeDB scalar types. Summed up, they are: `int16`, `int32`, `int64`, `float32`, `float64`, `bigint`, `decimal`, `sequence`, `str`, `bool`, `datetime`, `duration`, `cal::local_datetime`, `cal::local_date`, `cal::local_time`, `cal::relative_duration`, `cal::date_duration`, `uuid`, `json`, and `enum`. You can see the documentation for them {ref}`here <docs:ref_datamodel_scalar_types>`.

## Using `unless conflict on` + `else` + `update`

We put an `constraint exclusive` on `name` so that we won't be able to have two characters with the same name. The idea is that someone might see a character in the book and insert it, and then someone else would try to do the same. So this character named Johnny will work:

```edgeql
insert NPC {
  name := 'Johnny'
};
```

But if we try again we will get this error: `ERROR: ConstraintViolationError: name violates exclusivity constraint`

But sometimes just generating an error isn't enough - maybe we want something else to happen instead of just giving up. This is where `unless conflict on` comes in, followed by an `else` to explain what to do. `unless conflict on` is probably easier to explain through an example. Here's one that shows what we can do to either insert a `City` with its population, and to `update` the population if it already exists in the database.

Munich had a population of 230,023 in 1880, and five years later it was 261,023. So let's imagine we are updating the `City` data and some cities might exist, while others might not. The `insert` will look like this:

```edgeql
insert City {
  name := 'Munich',
  population := 261023,
} unless conflict on .name
else (
  update City
  set {
    population := 261023,
  }
);
```

Let's look at it step by step:

First a normal insert:

```edgeql
insert City {
  name := 'Munich'
}
```

But there might be a `City` there already so it might not work, so we add `unless conflict on .name`. Then follow it with an `else` to give instructions on what to do.

But here's the important part: we don't write `else insert`, because we are already in the middle of an `insert`. What we write instead is `update` for the type we are inserting, so `update City`. We are basically taking the failed data from the insert and updating it to try again. So we'll do this instead:

```edgeql
else (
  update City
  set {
    population := 261023
  }
);
```

With this, we are guaranteed to get a `City` object called Munich with a population of 261,023, whether it already exists in the database or not.

[Here is all our code so far up to Chapter 10.](code.md)

<!-- quiz-start -->

## Time to practice

1. Try inserting two `NPC` types in one insert with the following `name`, `first_appearance` and `last_appearance` information.

   `{('Jimmy the Bartender', '1887-09-10', '1887-09-11'), ('Some friend of Jonathan Harker', '1887-07-08', '1887-07-09')}`

2. Here are two more `NPC`s to insert, except the last one has an empty set (she's not dead). What problem are we going to have?

   `{('Dracula\'s Castle visitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05-08', {})}`

3. How would you order the `Person` types by last letter of their names?

4. Try inserting an `NPC` with the name `''`. Now how would you do the same query in question 3?

   Hint: the length of `''` is 0, which may be a problem.

5. Dr. Van Helsing has a list of `MinorVampire`s with their names and strengths. We already have some `MinorVampire`s in the database. How would you `insert` them while making sure to `update` if the object is already there?


[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Will they believe Van Helsing when he tells them the truth?_
