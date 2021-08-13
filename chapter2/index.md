---
tags: Scalar Types, Abstract Types, Filter
leadImage: illustration_02.jpg
---

# Chapter 2 - At the Hotel in Bistritz

We continue to read the story as we think about the database we need to store the information. The important information is in bold:

> Jonathan Harker has found a hotel in **Bistritz**, called the **Golden Krone Hotel**. He gets a welcome letter there from Dracula, who is waiting in his **castle**. Jonathan Harker will have to take a **horse-driven carriage** to get there tomorrow. We also see that Jonathan Harker is from **London**. The innkeeper at the Golden Krone Hotel seems very afraid of Dracula. He doesn't want Jonathan to leave and says it will be dangerous, but Jonathan doesn't listen. An old lady gives Jonathan a golden crucifix and says it will protect him. Jonathan is embarrassed, and takes it to be polite. Jonathan has no idea how much it will help him later.

Now we are starting to see some detail about the city. Reading the story, we see that we could add another property to `City`, and we will call it `important_places`. That's where places like the **Golden Krone Hotel** could go. We're not sure if the places will be their own types yet, so we'll just make it an array of strings: `property important_places -> array<str>;` We can put the names of important places in there and maybe develop it more later. It will now look like this:

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
```

Now our original insert for Bistritz will look like this:

```edgeql
INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

## Enums, scalar types, and extending

We now have two types of transport in the book: train, and horse-drawn carriage. The book is based in 1887, and our game will let the characters use types of transport that were available that year. Here an `enum` (enumeration) is probably the best choice, because an `enum` is about making one choice between options. The variants of the enum should be written in UpperCamelCase.

Here we see the word `scalar` for the first time: this is a `scalar type` because it only holds a single value at a time. The other types (`City`, `Person`) are `object types` because they can hold multiple values at the same time.

The other keyword we will see for the first time is `extending`, which means to take a type as a base and extend it. This gives you all the power of the type that you are extending, and adds some more options. We will write our `Transport` type like this:

```sdl
scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;
```

Did you notice that `scalar type` ends with a semicolon and the other types don't? That's because the other types have a `{}` to make a full expression. But here on a single line we don't have `{}` so we need the semicolon to show that the expression ends here.

To choose between the variants (the choices) in an enum, you just use a `.`. For our enum, that means we can choose `Transport.Feet`, `Transport.Train`, or `Transport.HorseDrawnCarriage`.

This `Transport` type is going to be for player characters in our game, not the people in the book (their stories and choices are already finished). That means that we will need a `PC` type and an `NPC` type, but our `Person` type should stay too - we can use it a base type for both. To do this, we can make `Person` an `abstract type` instead of just a `type`. Then with this abstract type, we can use the keyword `extending` for the other `PC` and `NPC` types.

So now this part of the schema looks like this:

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
}

type PC extending Person {
  required property transport -> Transport;
}

type NPC extending Person {
}
```

Now the characters from the book will be `NPC`s (non-player characters), while `PC` is being made with our game in mind. And because `Person` is now an abstract type, we can't insert it directly anymore. It will give us this error if we try to do something like `INSERT Person {name := 'Mr. HasAName'};`:

```
error: cannot insert into abstract object type 'default::Person'
  ┌─ query:1:8
  │
1 │ INSERT Person {name := 'Mr. HasAName'};
  │        ^^^^^^ error
```

No problem - just change `Person` to `NPC` and it will work.

Also, `SELECT` on an abstract type is just fine - it will select all the types that extend from it.

Let's also experiment with a player character. We'll make one called Emil Sinclair who starts out traveling by horse-drawn carriage. We'll also just give him `City` so he'll have all three cities.

```edgeql
INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := Transport.HorseDrawnCarriage,
};
```

Entering `places_visited := City` is short for `places_visited := (SELECT City)` - you don't have to type `SELECT` every time.

Note that we didn't just write `HorseDrawnCarriage`, because we have to choose the enum `Transport` and then make a choice of one of the variants.

## Casting 

Casting means to quickly change one type into another, Casting is used a lot in EdgeDB because it is strict about types, and will refuse to do operations on two types that are different. A lot of casting is done automatically out of convenience, such as with numbers. For example:

```edgeql
SELECT 9 + 9.9;
```

EdgeDB will not generate an error here and will just give the output of `18.9`, returning a `float64`. You can confirm that here:

```edgeql
SELECT (9 + 9.9) IS float64;
```

This will give `true`, while `SELECT (9 + 9.9) IS float32;` gives `false`.

When you need to do the cast yourself, you can indicate the type using `<>` angle brackets. For example, this will generate an error:

```edgeql
SELECT '9' + 9;
```

EdgeDB tells us the exact problem here:

```
error: operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
  ┌─ query:1:8
  │
1 │ SELECT '9' + 9;
  │        ^^^^^^^ Consider using an explicit type cast or a conversion function.

```

And to fix it, just use the angle brackets:

```
SELECT <int32>'9' + 9;
```

And you will get `18`, a 32-bit integer.

You can cast more than once at a time if you need to. This example isn't something you will need to do but shows how you can cast over and over again if you want:

```edgeql
SELECT <str><int64><str><int32>50 IS str;
```

That also gives us `{true}` because all we did is ask if it is a `str`, which it is.

Casting works from right to left, with the final cast on the far left. So `<str><int64><str><int32>50` means "50 into an int32 into a string into an int64 into a string". Or you can read it left to right like this: "A string from an int64 from a string from an int32 from the number 50".

Also note that casting is only for scalar types: user-created object types like `City` and `Person` are too complex to simply cast into each other.

## Filter

Finally, let's learn how to `FILTER` before we're done Chapter 2. You can use `FILTER` after the curly brackets in `SELECT` to only show certain results. Let's `FILTER` to only show `Person` types that have the name 'Emil Sinclair':

```edgeql
SELECT Person {
  name,
  places_visited: {name},
} FILTER .name = 'Emil Sinclair';
```

`FILTER .name` is short for `FILTER Person.name`. You can write `FILTER Person.name` too if you want - it's the same thing.

The output is this:

```
{
  default::PC {
    name: 'Emil Sinclair',
    places_visited: {
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
    },
  },
}
```

Let's filter the cities now. One flexible way to search is with `LIKE` or `ILIKE` to match on parts of a string.

- `LIKE` is case-sensitive: "Bistritz" matches "Bistritz" but "bistritz" does not.
- `ILIKE` is not case-sensitive (the I in ILIKE means **insensitive**), so "Bistritz" matches "BiStRitz", "bisTRITz", etc.

You can also add `%` on the left and/or right which means match anything before or after. Here are some examples with the matched part **in bold**:

- `LIKE Bistr%` matches "**Bistr**itz" (but not "bistritz"),
- `ILIKE '%IsTRiT%'` matches "B**istrit**z",
- `LIKE %athan Harker` matches "Jon**athan Harker**",
- `ILIKE %n h%` matches "Jonatha**n H**arker".

Let's `FILTER` to get all the cities that start with a capital B. That means we'll need `LIKE` because it's case-sensitive:

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name LIKE 'B%';
```

Here is the result:

```
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

You can also index a string with `[]` square brackets, starting at 0. For example, the indexes in the string 'Jonathan' look like this:

```
J o n a t h a n
0 1 2 3 4 5 6 7
```

So `'Jonathan'[0]` is 'J' and `'Jonathan'[4]` is 't'.

Let's try it:

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name[0] = 'B'; # First character must be 'B'
```

That gives the same result. Careful though: if you set the number too high then it will try to search outside of the string, which is an error. If we change 0 to 18 (`FILTER .name[18]; = 'B';`), we'll get this:

```
ERROR: InvalidValueError: string index 18 is out of bounds
```

Plus, if you have any `City` types with a name of `''`, even a search for index 0 will cause an error. 

You can also slice a string to get a piece of it. Because 'Jonathan' starts at zero, its index values look like this:

```
|J|o|n|a|t|h|a|n|
0 1 2 3 4 5 6 7 8
```

It's 8 characters long, so it fits entirely between 0 and 8. If you take a "slice" of it between indexes 2 and 5, you get 'nat' (`'Jonathan'[2:5]` = 'nat'), because it starts at 2 and goes *up to* 5 - but not including index 5. It's sort of like when you phone your friend to tell them that you're 'at their house': you're not telling them that you're inside it.

Negative index values are counted from the end of 'Jonathan', which is 8, so -1 corresponds to `8 - 1` (= 7), etc.

So what if you want to make sure that you won't get an error with an index number that might be too high? Here you can use `LIKE` or `ILIKE` with an empty parameter, because it will just give an empty set: `{}` instead of an error. `LIKE` and `ILIKE` are safer than indexing if there is a chance of having data that is too short in a property. There is a small lesson to be had here:

- "no data" in Edgedb is shown as an empty set: `{}`
- `""` (an empty string) is actually data.

Keeping that in mind helps you understand the behaviour between the two.


Finally, did you notice that we wrote a comment with `#` just now? Comments in EdgeDB are simple: anything to the right of `#` on a line gets ignored.

So this:

```edgeql
SELECT 1887#0503 is the first day of the book Dracula when...
;
```

returns `{1887}`.

[Here is all our code so far up to Chapter 2.](code.md)

<!-- quiz-start -->

## Time to practice

1. Change the following `SELECT` to display `{100}` by casting: `SELECT '99' + '1'`;
2. Select all the `City` types that start with 'Mu' (case sensitive).
3. Select the third letter (i.e. index number 2) of the name of every `NPC`.
4. Imagine an abstract type called `HasAString`:

   ```sdl
   abstract type HasAString {
     property string -> str
   };
   ```

   How would you change the `Person` type to extend `HasAString`?

5. This query only shows the id numbers of the places visited. How do you show their name?

   ```edgeql
   SELECT Person {
     places_visited
   };
   ```

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan gets into the carriage and travels into the cold mountains._
