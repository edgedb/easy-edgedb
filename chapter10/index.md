---
tags: Tuples, Computed Properties, Math
---

# Chapter 10 - Terrible events in Whitby

> Mina and Lucy are enjoying their time in Whitby. They spend a lot of time hiking nearby the coast and enjoying the view from the ruins of Whitby Abbey, an old church from long ago. One night there is a huge storm and a ship arrives in the fog - it's the Demeter, carrying Dracula. Lucy later begins to sleepwalk at night and looks very pale, and always says strange things. Mina tries to stop her, but sometimes Lucy gets outside.
>
> One night Lucy watches the sun go down and says: "His red eyes again! They are just the same." Mina is worried and asks Dr. Seward for help, who examines Lucy. She is pale and weak, but Dr. Seward doesn't know why. He decides to call his old teacher Abraham Van Helsing, who comes from the Netherlands to help. Van Helsing examines Lucy and looks shocked. Then he turns to the others and says, "Listen. We can help this girl, but you are going to find the methods very strange. You are going to have to trust me..."

The city of Whitby is in the northeast of England. Right now our `City` type just extends `Place`, which only gives us the properties `name`, `modern_name` and `important_places`. This could be a good time to give it a `population` property which can help us draw the cities in our game. It will be an `int64` to give us the size we need:

```sdl
type City extending Place {
  population: int64;
}
```

By the way, here are the approximate populations for our five cities at the time of the book. They were much smaller back in 1893:

- Buda-Pesth (Budapest): 402706
- London: 3500000
- Munich: 230023
- Whitby: 14400
- Bistritz (Bistrița): 9100

Now let's do a migration.

Whitby is the only one of the five that isn't in our database already. Inserting it is easy enough:

```edgeql
insert City {
  name := 'Whitby',
  population := 14400,
  important_places := ['Whitby Abbey']
};
```

But for the rest of them it would be nice to update everything at the same time.

## Working with tuples and arrays

If we have all the city data together, we can do a single insert with a `for` and `union` loop again. Let's imagine that the city data we have inside tuples, which seem similar to arrays but are quite different. One big difference is that a tuple can hold different types, so this is okay:

```
('Buda-Pesth', 402706), ('London', 3500000), 
('Munich', 230023),     ('Bistritz', 9100)
```

In this case, the type is called a `tuple<str, int64>`.

Before we start using these tuples, let's make sure that we understand the difference between tuples and arrays. To start, let's look at slicing arrays and strings in a bit more detail.

Previously we learned how to use square brackets to access part of an array or a string. So this query:

```edgeql
select ['Mina Murray', 'Lucy Westenra'][1];
```

will give the output `{'Lucy Westenra'}` (that's index number 1).

We also learned that we can use a colon to indicate the starting and ending index, like in this example:

```edgeql
select NPC.name[0:10];
```

The output shows the first ten letters of every NPC's name:

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

This prints from index 2 up to 2 indexes away from the end (in other words, it'll cut off two letters from each side). Here's the output:

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

Tuples are very different. You can think of them as similar to object types with properties that are numbered instead of named. This is why tuples can hold different types together: `str` with `array<bool>`, `int64` with `float32`, you name it.

So this is completely fine:

```edgeql
select {
  ('Bistritz', 9100, cal::to_local_date(1893, 5, 6)),
  ('Munich', 230023, cal::to_local_date(1893, 5, 8))
};
```

The output is:

```
{
  ('Bistritz', 9100, <cal::local_date>'1893-05-06'),
  ('Munich', 230023, <cal::local_date>'1893-05-08'),
}
```

You can really see how similar tuples are to object types by doing a query on one of their properties. Here is the same query as above, except that we will select property `.0` instead of the whole set:

```edgedb
select {
  ('Bistritz', 9100, cal::to_local_date(1893, 5, 6)),
  ('Munich', 230023, cal::to_local_date(1893, 5, 8))
}.0; # Only the .0 is different from the query above
```

The output is `{'Bistritz', 'Munich'}`, so pretty much the same as doing a `select City.name`;

Tuples can hold multiple types (this one is type `tuple<str, int64, cal::local_date>`), but you can't work with tuples of different types. So this is not allowed:

```edgeql
select {(1, 2, 3), (4, 5, '6')};
```

EdgeDB will give an error because it won't try to work with tuples that are of different types. It complains:

```
error: InvalidTypeError: set constructor has arguments of incompatible types 
'tuple<std::int64, std::int64, std::int64>' and 
'tuple<std::int64, std::int64, std::str>'
  ┌─ <query>:1:8
  │
1 │ select {(1, 2, 3), (4, 5, '6')};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast 
  or a conversion function.
```

You'll notice that the error suggests that we cast one of the items inside one of the tuples to match the other. Doing so removes the error and EdgeDB is happy again:

```edgeql
select {(1, 2, 3), (4, 5, <int64>'6')};
```

Now that we know all this, we can update all our cities at the same time. It looks like this:

```edgeql
for data in {('Buda-Pesth', 402706), ('London', 3500000),
  ('Munich', 230023), ('Bistritz', 9100)}
union (
  update City filter .name = data.0
  set {
    population := data.1
  }
);
```

So this query accesses each tuple one at a time in the `for` loop, filters by the string (which is `data.0`) and then updates with the population (which is `data.1`).

You can actually choose to give names to the items inside tuples if you like. This makes them feel even more like the object types in our schema. Here are the same cities except now we can access them by name:

```edgeql
with cities := 
(
  (name := 'Buda-Pesth', pop := 402706), 
  (name := 'London',     pop := 3500000), 
  (name := 'Munich',     pop := 230023),
  (name := 'Bistritz',   pop := 9100)
),
  select cities.1.pop;
```

This returns `{3500000}`, the population of London.

Similarly, we can give each of the tuples inside the `cities` tuple a name too!

```edgeql
with cities := 
(
  budapest := (name := 'Buda-Pesth', pop := 402706), 
  london   := (name := 'London',     pop := 3500000), 
  munich   := (name := 'Munich',     pop := 230023),
  bistritz := (name := 'Bistritz',   pop := 9100)
  ),
  select cities.munich.pop;
```

Now we get `{230023}`, the population of Munich.

You can still access items inside tuples by numbers even if they have a name:

```
db> select (name := 'Jonathan Harker', age := 25).0;
{'Jonathan Harker'}
db> select (name := 'Jonathan Harker', age := 25).name;
{'Jonathan Harker'}
```

And also note that if you choose to name the items inside a tuple you have to name them all. So this won't work:

```edgeql
select ('Jonathan Harker', age := 25).age;
```

Let's finish this section with a final note about casting. We know that we can cast into any scalar type, and this works for tuples of scalar types too. It uses the same format with `<>` except that you put it inside of `<tuple>`. This is a convenient way to do multiple casts at the same time. Take this query for example:

```edgeql
with london := ('London', 3500000),
select <tuple<json, int32>>london;
```

Using `<tuple<json, int32>>` lets us cast the whole tuple instead of doing a cast for each individual type inside the tuple.

That gives us this output:

```
{(Json("\"London\""), 3500000)}
```

Here's another example if we need to do some math with floats to calculate an increase in London's population:

```edgeql
with 
  london := ('London', 3500000),
  # Cast into a float so we can do some precise math
  float_london := <tuple<str, float64>>(london),
  # Increase population, cast back for readability
  select <tuple<str, int32>>(float_london.0, float_london.1 * 1.035);
```

The output is `{('London', 3622500)}`.

## More on ordering and using math

Now that we have some numbers, we can start playing around with ordering and math. We tried out ordering for the first time in Chapter 7 and it was quite simple: type `order by` and then indicate the property/link you want to order by. Here we order them by population:

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

For some actual math, you can check out the functions in `std` {eql:func}`here <docs:std::sum>` as well as the `math` module {ref}`here <docs:ref_std_math>`. Instead of looking at each function separately, let's do a single big query to show many of them together. To make the output nice, we will write it together with strings explaining the results and then cast them all to `<str>` so we can join them together using `++`.

```edgeql
with pop := City.population
select (
  'Number of cities with population data: ' ++ <str>count(pop),
  'All cities have more than 50,000 people: ' ++ <str>all(pop > 50000),
  'Total population: ' ++ <str>sum(pop),
  'Smallest/largest population: ' ++ <str>min(pop) ++ ', ' ++ <str>max(pop),
  'Average population: ' ++ <str>math::mean(pop),
  'Any cities with more than 5 million people? ' ++ <str>any(pop > 5000000),
  'Standard deviation: ' ++ <str>math::stddev(pop)
);
```

This query used quite a few functions, all of which work on sets:

- `count()` to count the number of items,
- `all()` to return `{true}` if all items match and `{false}` otherwise,
- `sum()` to add them all together,
- `max()` to return the highest value,
- `min()` to return the lowest value,
- `math::mean()` to give the average,
- `any()` to return `{true}` if any item matches and `{false}` otherwise, and
- `math::stddev()` for the standard deviation.

The output also makes it clear how they work:

```
{
  (
    'Number of cities with population data: 5',
    'All cities have more than 50,000 people: false',
    'Total population: 4156229',
    'Smallest/largest population: 9100, 3500000',
    'Average population: 831245.8',
    'Any cities with more than 5 million people? false',
    'Standard deviation: 1500876.8248',
  ),
}
```

`any()`, `all()` and `count()` are particularly useful in operations to give you an idea of your data.

## Importing modules using the `with` keyword

You can use the `with` keyword to import modules too. In the example above we used two functions from EdgeDB's `math` module: `math::mean()` and `math::stddev()`. Just writing `mean()` and `stddev()` would produce this error:

```
edgedb error: InvalidReferenceError: function 'default::mean' does not exist
```

If you don't want to write the module name every time you can just import the module after `with`. Let's slip that into the query we just used. See if you can see what's changed:

```edgeql
with 
  pop := City.population,
  module math,
select (
  'Number of cities with population data: ' ++ <str>count(pop),
  'All cities have more than 50,000 people: ' ++ <str>all(pop > 50000),
  'Total population: ' ++ <str>sum(pop),
  'Smallest/largest population: ' ++ <str>min(pop) ++ ', ' ++ <str>max(pop),
  'Average population: ' ++ <str>mean(pop),
  'Any cities with more than 5 million people? ' ++ <str>any(pop > 5000000),
  'Standard deviation: ' ++ <str>stddev(pop)
);
```

The output is the same, but we added an import of the `math` module, letting us just write `mean()` and `stddev()`.

You can also use `as` to rename a module (well, to _alias_ a module) in the same way that you can rename a type. So this will work too:

```edgeql
with M as module math,
select M::mean(City.population);
```

That gives us the mean: `{831245.8}`.

You can also set a module as the default just by typing `set module` followed by the module name. This will unset the default module though. So the same query as above would look like this:

```
db> set module math;
OK: SET ALIAS
# Note: default::City instead of just City
db> select mean(default::City.population);
```

This makes it easy to have separate modules (`module test` for example with test types and data) that you can quickly switch to and query without needing to type too much.

## Some more computed properties for names

We saw in this chapter that Dr. Seward asked his old teacher Dr. Van Helsing to come and help Lucy. Here is how Dr. Van Helsing began his letter to say that he was coming:

```
Letter, Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc., to Dr. Seward.

“2 September.

“My good Friend,—
“When I have received your letter I am already coming to you.
```

The `Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc.` part is interesting. This might be a good time to think more about the `name` property inside our `Person` type. Right now our `name` property is just a single string, but we have people with different types of names, in this order:

Title | First name | Last name | Degree

Here are some examples:

* 'Count Dracula' (title + name),
* 'Dr. Seward' (title + name),
* 'John Seward, M.D.' (name + degree),
* 'Mr. Renfield' (title + name),
* 'Dr. Abraham Van Helsing, M.D, Ph. D. Lit.' (title + first name + last name + degrees),
* 'Lord Godalming' (title + name)

That would lead us to think that we should have properties like `first_name`, `last_name`, and `title` and then join them together using a computed property. But then again, not every character has these exact four parts to their name. Some others that don't are 'Vampire Woman 1' and 'The Innkeeper', and our game would certainly have a lot more of these. It's also somewhat rare to use all four of these properties together: Van Helsing's friends call him "Doctor Van Helsing", not "Dr. Abraham Van Helsing, M.D, Ph. D. Lit."!

So it's probably not a good idea to get rid of `name` and to always build names from separate parts. But in our game we might have characters writing letters or talking to each other, and they will have to use things like titles and degrees.

We could try a middle of the road approach for our `Person` type instead. We'll keep `name`, and add some computed properties below it. The property `degrees` will be an `array<str>`. We can then use the `array_join()` function to join them together. This function takes an array, plus a string called a `delimeter` to tell the function what to place in between each item in the array.

Here are two quick examples of `array_join()`:

```
# No delimiter, so just joins the two strings
db> select array_join(['Jonathan ', 'Harker'], '');
{'Jonathan Harker'}
# Delimiter of comma and space
db> select array_join(['And a one', 'and a two', 'and a three'], ', ');
{'And a one, and a two, and a three'}
# Without the delimiter:
db> select array_join(['And a one', 'and a two', 'and a three'], '');
{'And a oneand a twoand a three'}
```

Now here is the `Person` type with its new properties:

```sdl
abstract type Person {
  required name: str {
    delegated constraint exclusive;
  }
  multi places_visited: Place;
  multi lovers: Person;
  property is_single := not exists .lovers;
  strength: int16;
  first_appearance: cal::local_date;
  last_appearance: cal::local_date;
  age: int16;
  title: str;
  degrees: array<str>;
  property conversational_name := .title ++ ' ' 
    ++ .name if exists .title else .name;
  property pen_name := .name ++ ', ' 
     ++ array_join(.degrees, ', ') if exists .degrees else .name;
}
```

Let's do a migration now, and try an insert for Van Helsing...or rather, Dr. Van Helsing!

```edgeql
insert NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := ['M.D.', 'Ph. D. Lit.', 'etc.']
};
```

John Seward is a doctor too so let's be sure to update him with a proper title and degree.

```edgeql
update NPC filter .name = 'John Seward'
set { 
  title := 'Dr.',
  degrees := ['M.D.']
};
```

Now we can make use of these properties to liven up our conversation engine in the game. For example:

```edgeql
with educated := (select Person filter exists .title and exists .degrees)
  select (
  'There goes ' ++ educated.name ++ '.',
  'I say! Are you ' ++ educated.conversational_name ++ '?',
  'I have a letter from you signed as follows:\n\t'
     ++ educated.pen_name);
```

By the way, the `\n` inside the string creates a *new* line, while `\t` moves it one *tab* to the right.

This gives us:

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'I have a letter from you signed as follows:
        Abraham Van Helsing, M.D., Ph. D. Lit., etc.',
  ),
  (
    'There goes John Seward.',
    'I say! Are you Dr. John Seward?',
    'I have a letter from you signed as follows:
        John Seward, M.D.',
  ),
}
```

If this were just a standard database with website users it would be much simpler: get users to enter their first names and last names and then use these two properties to compute a full name. But the setting in Bram Stoker's Dracula is much more complex than that!

## Other escape characters and raw strings

Besides `\n` and `\t` there are quite a few other escape characters - you can see the complete list {ref}`here <docs:ref_eql_lexical_str_escapes>`. Some are rare but hexadecimal with `\x` and unicode escape character with `\u` are two that might be useful.

For a quick example of unicode escape characters, try pasting this into your REPL to decode it into what Van Helsing had to say during his first visit.

```edgeql
select '\u004E\u0061\u0079\u002C\u0020\u0049\u0020\u0061\u006D
\u006E\u006F\u0074\u0020\u006A\u0065\u0073\u0074\u0069\u006E\u0067\u002E
\u0054\u0068\u0069\u0073\u0020\u0069\u0073\u0020\u006E\u006F
\u006A\u0065\u0073\u0074\u002C\u0020\u0062\u0075\u0074
\u006C\u0069\u0066\u0065\u0020\u0061\u006E\u0064\u0020\u0064\u0065\u0061\u0074\u0068\u002C
\u0070\u0065\u0072\u0068\u0061\u0070\u0073\u0020\u006D\u006F\u0072\u0065\u002E\u2019';
```

If you want to ignore escape characters, put an `r` (which stands for _raw_) in front of the quote. Let's try it with the example above. Only the last part has an `r`:

```edgeql
with educated := (select Person filter exists .title and exists .degrees)
  select (
  'I say! Are you ' ++ educated.conversational_name ++ '?',
  r'I have a letter from you signed as follows:\n\t'
     ++ educated.pen_name);
```

Now we get:

```
{
  (
    'I say! Are you Dr. Abraham Van Helsing?',
    'I have a letter from you signed as follows:\\n\\tAbraham Van Helsing, M.D., Ph. D. Lit., etc.',
  ),
  (
    'I say! Are you Dr. John Seward?',
    'I have a letter from you signed as follows:\\n\\tJohn Seward, M.D.',
  ),
}
```

Finally, EdgeDB can also use `$$` to make raw string literals. Any string inside this will ignore any and all quotation marks and escape characters, so you won't have to worry about the string ending in the middle. Here's one example with a bunch of single and double quotes inside:

```edgeql
select $$ 
"Dr. Van Helsing would like to tell "them" 
about "vampires" and how to "kill" them, 
but he'd sound crazy."
$$;
```

Without the `$$` EdgeDB would generate an error as it would treat the input as four separate strings with three unknown keywords between them. From EdgeDB's point of view the input would look like this:

```
"Dr. Van Helsing would like to tell "
them
" about "
vampires
" and how to "
kill
" them, but he'd sound crazy."
```

## Using `unless conflict on` + `else` + `update`

We have an `constraint exclusive` on `name` so that we won't be able to have two characters with the same name. The idea is that someone might see a character in the book and insert it, and then someone else would try to do the same. So this character named Johnny will work:

```edgeql
insert NPC {
  name := 'Johnny'
};
```

But if we try again we will get this error:

```
edgedb error: ConstraintViolationError: name violates exclusivity constraint
```

But sometimes just generating an error isn't enough - maybe we want something else to happen instead of just giving up. This is where `unless conflict on` comes in, followed by an `else` to explain what to do to the existing object.

`unless conflict on` is easiest to explain through an example. We've already populated our database with some city data that comes from the year 1880, but what if we came across some data for 1885 instead which is closer to the setting in the book? Larger cities have better items, more NPCs and quests to do in our game, so having an accurate population is important. But we can't just use `insert` everywhere, because cities like Munich are already in the database. So this insert would just generate an error and give up:

```edgeql
# Munich had a population of 230,023 in 1880 and 261,023 in 1885
insert City {
  name := 'Munich',
  population := 261023
};
```

However, we also can't just `update` every `City` object either, because a lot of the cities in the 1885 data aren't in the 1880 data - they are new cities. In this case we would like to *try* to insert a new `City` object. But if the object already exists, then update its population instead of just giving up.

The way to accomplish this is by first trying an insert, then following with `unless conflict on`, `else` and `update`.

Here is how we would do it for Munich:

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

Here we tell EdgeDB to keep an eye out for any conflicts by using `unless conflict on .name`, followed by `else` to give instructions on what to do to the existing object in the database. Also note that we don't write `else insert`, because the conflict means that we are unable to do an `insert`. What we write instead is `update` for the conflicting object that is already in the database: `update City`.

With this, we are guaranteed to get a `City` object called Munich with a population of 261,023, whether it already exists in the database or not.

[Here is all our code so far up to Chapter 10.](code.md)

<!-- quiz-start -->

## Time to practice

1. Try inserting two `NPC` types in one insert with the following `name`, `first_appearance` and `last_appearance` information.

   `{('Jimmy the Bartender', '1893-09-10', '1893-09-11'), ('Some friend of Jonathan Harker', '1893-07-08', '1893-07-09')}`

2. Here are two more `NPC`s to insert, except the last one has an empty set (she's not dead). What problem are we going to have?

   `{('Dracula\'s Castle visitor', '1893-09-10', '1893-09-11'), ('Old lady from Bistritz', '1893-05-08', {})}`

3. How would you order the `Person` types by last letter of their names?

4. Try inserting an `NPC` with the name `''`. Now how would you do the same query in question 3?

   Hint: the length of `''` is 0, which may be a problem.

5. Dr. Van Helsing has a list of `MinorVampire`s with their names and strengths. We already have some `MinorVampire`s in the database. How would you `insert` them while making sure to `update` if the object is already there?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Will they believe Van Helsing when he tells them the truth?_
