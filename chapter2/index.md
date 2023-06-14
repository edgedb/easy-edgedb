---
tags: Scalar Types, Abstract Types, Filtering, Slicing
leadImage: illustration_02.jpg
---

# Chapter 2 - At the Hotel in Bistritz

We continue to read the story as we think about the database we need to store the information. The important information is in bold:

> Jonathan Harker has found a hotel in **Bistritz**, called the **Golden Krone Hotel**. He gets a welcome letter there from Dracula, who is waiting in his **castle**. Jonathan Harker will have to take a horse-driven carriage to get there tomorrow. Jonathan Harker is originally from **London**. The innkeeper at the Golden Krone Hotel seems very afraid of Dracula. He doesn't want Jonathan to leave and says it will be dangerous, but Jonathan doesn't listen. An old lady gives Jonathan a golden crucifix and says that it will protect him. Jonathan is embarrassed, and takes it to be polite. Jonathan has no idea how much it will help him later.

Now we are starting to see some detail about the city. Reading the story, we see that we could add another property to `City`, and we will call it `important_places`. That's where places like the **Golden Krone Hotel** could go. We're not sure if the places will be their own types yet, so we'll just make it an array of strings: `important_places: array<str>;` We can put the names of important places in there and maybe develop it more later. It will now look like this:

```sdl
type City {
  required name: str;
  modern_name: str;
  important_places: array<str>;
}
```

Now our insert for Bistritz will look like this:

```edgeql
insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

After this insert, our database will have two cities called Bistritz. To delete one of them, we will need to know two more keywords: `filter` and `delete`. We will learn `filter` in this chapter, and `delete` in the next, and then we will delete our duplicate Bistritz.

## Enums, scalar types, and extending

Our game is going to need to have player characters in it, which means that users of the game are going to have to choose what type of character they want to be. The book is [probably based in 1893](https://vampvault.jimdofree.com/tidbits/timeframe/), and our game will let the characters use classes that make sense for this time period. Here an `enum` (enumeration) is probably the best choice, because an `enum` is about choosing one of several options. The values of the enum should be written in UpperCamelCase.

Here we see the word `scalar` for the first time: this is a `scalar type`, which means that it only holds a single value at a time. The other types (`City`, `NPC`) are `object types` because they can hold multiple values at the same time.

The other keyword we will see for the first time is `extending`, which means to take a type as a base and extend it. This keyword gives you all the power of the type that you are extending, and lets you add more on top. There will probably be more character classes in a real game, but for the moment we will write our `Class` type like this with three classes:

```sdl
scalar type Class extending enum<Rogue, Mystic, Merchant>;
```

Did you notice that `scalar type` ends with a semicolon and the other types don't? That's because the other types have a `{}` to make a full expression. But here on a single line we don't have `{}` so we need the semicolon to show that the expression ends here.

To choose between the values (the choices) in an enum, you just use a `.`. For our enum, that means we can choose `Class.Rogue`, `Class.Mystic`, or `Class.Merchant`.

This `Class` type is going to be for player characters in our game, not the people in the book (their stories, choices and statistics are already decided). That means that we will need both an `PC` type and an `NPC` type. These two types are pretty similar to each other: they both have a `name`, `places_visited`, and probably  some other properties that we will add later. This is a good case for an abstract type. Inside this abstract type we can put all the properties that are common to `PC` and `NPC`, which will use the `extending` keyword to gain the properties that `Person` has.

So now this part of the schema looks like this:

```sdl
abstract type Person {
  required name: str;
  multi places_visited: City;
}

type PC extending Person {
  required class: Class;
}

type NPC extending Person {
}
```

The characters from the book are still `NPC`s (non-player characters), while `PC` is being made with our game in mind. And because `Person` is an abstract type, we can't insert directly anymore. It will give us this error if we try to do something like `insert Person {name := 'Mr. HasAName'};`:

```
error: QueryError: cannot insert into abstract object type 'default::Person'
  ┌─ <query>:1:8
  │
1 │ insert Person {name := 'Mr. HasAName'};
  │        ^^^^^^ error
```

No problem - just change `Person` to `NPC` and it will work.

However, `select` on an abstract type is just fine - it will select all the types that extend from it. Let's add a `PC` object now to make a `select` on the abstract `Person` type more interesting. 

We'll make a `PC` called Emil Sinclair who is a mystic. We'll also just give him `City` so he'll have all three cities.

```edgeql
insert PC {
  name := 'Emil Sinclair',
  places_visited := City,
  class := Class.Mystic,
};
```

Entering `places_visited := City` is short for `places_visited := (select City)` - you don't have to type `select` every time. Also note that we didn't just write `Mystic`, because we have to choose the enum `Class` and then make a choice of one of the values.

However, EdgeDB also allows you to choose an enum just by writing a string that matches the value. So this insert would have worked too:

```edgeql
insert PC {
  name := 'Emil Sinclair',
  places_visited := City,
  class := 'Mystic',
};
```

But if you were to type `class := 'wrestler'` it wouldn't work:

```
edgedb error: InvalidValueError: invalid input value for enum 'default::Class': "wrestler"
```

Now let's do a select on the abstract type `Person` to see the objects inside, plus the properties common to both: `name` and `places_visited`.

```edgeql
select Person {
  name,
  places_visited: {
    name
  }
};
```

This gives us an output showing objects of both the `PC` and `NPC` type.

```
{
  default::NPC {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
    },
  },
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

## Casting 

Casting means to quickly change one type into another. Casting is used a lot in EdgeDB because it is strict about types, and will refuse to do operations on two types that are different. A lot of casting is done automatically out of convenience, such as with numbers. For example:

```edgeql
select 9 + 9.9;
```

The 9 in this example is an `int64`, while 9.9 is a `float64`. And though 9 and 9.9 are different types, EdgeDB will not generate an error here and will just give the output of `18.9`, returning a `float64`. You can confirm that here:

```edgeql
select (9 + 9.9) is float64;
```

This will give `true`, while `select (9 + 9.9) is float32;` gives `false`.

When a cast is done without you needing to type anything extra, it is known as an _implicit cast_.

When you need to do the cast yourself, it is called an _explicit cast_. In this case you can indicate the type using `<>` angle brackets. For example, this will generate an error:

```edgeql
select '9' + 9;
```

EdgeDB tells us the exact problem here:

```
error: InvalidTypeError: operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
  ┌─ <query>:1:8
  │
1 │ select '9' + 9;
  │        ^^^^^^^ Consider using an explicit type cast or a conversion function.
```

And to fix it, just use the angle brackets:

```edgeql
select <int64>'9' + 9;
```

And you will get `18`, a 64-bit integer.

Of course, a cast won't work if the input is invalid:

```edgeql
db> select <int64>"Hi I'm a number please add me to" + 9;
edgedb error: InvalidValueError: invalid input syntax for type std::int64: "Hi I'm a number please add me to"
```

You can cast more than once at a time if you need to. This example isn't something you will need to do but shows how you can cast over and over again if you want:

```edgeql
select <str><int64><str><int32>50 is str;
```

That also gives us `{true}` because all we did is ask if it is a `str`, which it is.

Casting works from right to left, with the final cast on the far left. Take the following example with a lot of casting:

```edgeql
select <str><int64><str><int32>50;
```

You can read it as "50 into an int32 into a string into an int64 into a string". Or you can read it left to right like this: "A string from an int64 from a string from an int32 from the number 50".

Also note that casting is only for scalar types: user-created object types like `City` and `Person` are too complex to simply cast into each other.

## Filter

To finish up the second chapter, let's learn how to `filter`. You can use `filter` after the curly brackets in `select` to only show certain objects instead of showing every object for a type. Let's `filter` to only show `Person` types that have the name 'Emil Sinclair':

```edgeql
select Person {
  name,
  places_visited: {name},
} filter .name = 'Emil Sinclair';
```

`filter .name` is short for `filter Person.name`. You can write `filter Person.name` too if you want - it's the same thing.

The output now only shows a single `PC` object with a name that matches our filter:

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

Let's filter the cities now. One flexible way to search is with `like` or `ilike` which allow us to to match on parts of a string.

- `like` is case sensitive: "Bistritz" matches "Bistritz" but "bistritz" does not.
- `ilike` is not case sensitive (the i in ilike means **insensitive**), so "Bistritz" matches both "BiStRitz" and "bisTRITz".

You can also add `%` on the left and/or right which means match anything before or after. Here are some examples with the matched part **in bold**:

- `like Bistr%` matches "**Bistr**itz" (but not "bistritz"),
- `ilike '%IsTRiT%'` matches "B**istrit**z",
- `like %athan Harker` matches "Jon**athan Harker**",
- `ilike %n h%` matches "Jonatha**n H**arker".

Let's `filter` to get all the cities that start with a capital B. That means we'll need `like` because it's case-sensitive:

```edgeql
select City {
  name,
  modern_name,
} filter .name like 'B%';
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
select City {
  name,
  modern_name,
} filter .name[0] = 'B'; # First character must be 'B'
```

That gives the same result. Careful though: if you set the number too high then it will try to search outside of the string, which is an error. If we change 0 to 18 (`filter .name[18] = 'B';`), we'll get this error:

```
edgedb error: InvalidValueError: string index 18 is out of bounds (on line 4, column 16)
```

Plus, if you have any `City` types with a name of `''`, even a search for index 0 will cause an error. 

Let's try to make that error happen.

First we will insert a `City` object with '' for a name:

```edgeql
db> insert City {
 name := ''
 };
{default::City {id: 5d01e634-f2c7-11ed-87af-d7dfced50628}}
```

And now our former query doesn't work, because EdgeDB will come across a `City` object with a name property that it can't index into.

```edgeql
db> select City {
  name,
  modern_name,
 } filter .name[0] = 'B';
edgedb error: InvalidValueError: string index 0 is out of bounds (on line 4, column 16)
```

So a good rule of thumb is to not use raw indexes when filtering unless you are sure that there will be a value, and that the value will be long enough. In the next chapter we will learn how to use _constraints_ to ensure that your data matches certain expectations you have, such as minimum length.

So what if you want to make sure that you won't get an error with an index number that might be too high? Here you can use `like` or `ilike` instead. If you replace the `.name[0]` part in the query above with `.name ilike 'B%'` we don't get an error, and the query still checks to see if there is a 'B' at index 0.

```edgeql
db> select City {
  name,
  modern_name,
 } filter .name ilike 'B%';
```

The result is now all `City` objects that start with 'B', and the `City' object with a `name` of `''` is no longer getting in the way.

 {
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}

## Slicing

You can also slice a string to get a piece of it. Let's look at the index values for 'Jonathan' again and think about how we could slice it.

```
J o n a t h a n
0 1 2 3 4 5 6 7
```

Slices represent a part of a string that starts at one index and ends _before_ another one. In other words, if you take a "slice" of it between indexes 2 and 5, you get 'nat' (`'Jonathan'[2:5]` = 'nat'), because it starts at 2 and goes *up to* 5 - but not including index 5. It's sort of like when you phone your friend to tell them that you're 'at their house': you're not telling them that you're inside it.

In the same way, selecting a slice of 'Jonathan' up to index 7 will show up to and including index 6:

```edgeql
db> select 'Jonathan'[0:7];
{'Jonatha'}
```

In this case, you could search up to index 8. You could even search up to index 1000, as slicing doesn't involve trying to directly access an index so it won't generate an error if there is no value at that index. Or you can use `[0:]` for an open-ended slice that starts at 0 and ends when the string ends. So all three of these queries will work without generating an error:

```edgeql
db> select 'Jonathan'[0:8];
{'Jonathan'}
db> select 'Jonathan'[0:1000];
{'Jonathan'}
db> select 'Jonathan'[0:];
{'Jonathan'}
```

This also means that you can use slicing to safely search for values at a certain index even if the value might be an empty string. At the moment we have a `City` object in the database with a `name` of `''`, so `select City.name[0]` generates an error:

```edgeql
db> select City.name[0];
edgedb error: InvalidValueError: string index 0 is out of bounds (on line 1, column 18)
```

However, if we change the query to `db> select City.name[0:1];` then it will look for a slice instead of an exact character index, and the query will work:

```
db> select City.name[0:1];
{'M', 'B', 'B', ''}
```

You can also use negative numbers to slice from the other end of a string. Negative index values are counted from the end of 'Jonathan', which is 8, so -1 corresponds to `8 - 1`: index number 7. Let's prove this with a query:

```edgeql
db> select {'Jonathan'[7] = 'Jonathan'[-1]};
{true}
```

Let's end the chapter with a quick note. Did you notice one of the queries used `#` to add a comment? Comments in EdgeDB are simple: anything to the right of `#` on a line gets ignored.

So this:

```edgeql
select 1893#0503 is the first day of the book Dracula when...
;
```

returns `{1893}`.

[Here is all our code so far up to Chapter 2.](code.md)

<!-- quiz-start -->

## Time to practice

1. Change the following `select` to display `{100}` by casting: `select '99' + '1'`;
2. Select all the `City` types that start with 'Mu' (case sensitive).
3. Select the third letter (i.e. index number 2) of the name of every `NPC`.
4. If some `NPC` objects might have a name of `''`, how would you find the third letter of the name of every `NPC`?
5. Imagine an abstract type called `HasAString`:

   ```sdl
   abstract type HasAString {
     string: str
   };
   ```

   How would you change the `Person` type to extend `HasAString`?

6. This query only shows the id numbers of the places visited. How do you show their name?

   ```edgeql
   select Person {
     places_visited
   };
   ```

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan gets into the carriage and travels into the cold mountains._
