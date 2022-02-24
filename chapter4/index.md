---
tags: Local Time, Advanced Filtering
leadImage: illustration_04.jpg
---

# Chapter 4 - "What a strange man this Count Dracula is."

> Jonathan Harker wakes up late and is alone in the castle. Dracula appears after nightfall and they talk **through the night**. Dracula is making plans to move to London, and Jonathan gives him some advice about buying houses. Jonathan tells Dracula that a big house called Carfax would be a good house to buy. It's very big and quiet. It's close to an asylum for crazy people, but not too close. Dracula likes the idea. He then tells Jonathan not to go into any of the locked rooms in the castle, because it could be dangerous. Jonathan sees that it's almost morning - they talked through the whole night again. Dracula suddenly stands up and says he must go, and leaves the room. Jonathan thinks about **Mina** back in London, who he is going to marry when he returns. He is beginning to feel that there is something wrong with Dracula, and the castle. Seriously, where are the other people?

First let's create Jonathan's girlfriend, Mina Murray. But we'll also add a new link to the `Person` type in the schema called `lover`:

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  link lover -> Person;
}
```

With this we can link the two of them together. We will assume that a person can only have one `lover`, so this is a `single link` but we can just write `link`.

Mina is in London, and we don't know if she has been anywhere else. So let's do a quick insert to create the city of London. It couldn't be easier:

```edgeql
INSERT City {
    name := 'London',
};
```

To give her the city of London, we can just do a quick `SELECT City FILTER .name = 'London'`. This will give her the `City` that matches `.name = 'London'`, but it won't give an error if the city's not there: it will just return a `{}` empty set.

## DETACHED, EXISTS, and LIMIT

For `lover` it is the same process but a bit more complicated:

```edgeql
INSERT NPC {
  name := 'Mina Murray',
  lover := assert_single(
    (
      SELECT DETACHED NPC
      FILTER .name = 'Jonathan Harker'
    )
  ),
  places_visited := (SELECT City FILTER .name = 'London'),
};
```

You'll notice two things here:

- `DETACHED`. This is because we are inside of an `INSERT` for the `NPC` type, but we want to link to the same type: another `NPC`. We need to add `DETACHED` to specify that we are talking about `NPC` in general, not the `NPC` that we are inserting right now.
- {eql:func}`docs:std::assert_single`. This is because the link is a `single link`. EdgeDB doesn't know how many results we might get: for all it knows, there might be 2 or 3 or more `Jonathan Harkers`. To guarantee that we are only creating a `single link`, we use `assert_single()` function.

Now we want to make a query to see who is single and who is not. This is easy by using a "computed" property, where we can create a new variable that we define with `:=`. First here is a normal query:

```edgeql
SELECT Person {
  name,
  lover: {
    name
  }
};
```

This gives us:

```
{
  default::NPC {name: 'Jonathan Harker', lover: {}},
  default::NPC {name: 'The innkeeper', lover: {}},
  default::NPC {name: 'Mina Murray', lover: default::NPC {name: 'Jonathan Harker'}},
  default::PC {name: 'Emil Sinclair', lover: {}},
  default::Vampire {name: 'Count Dracula', lover: {}},
}
```

Okay, so Mina Murray has a lover but Jonathan Harker does not yet, because he was inserted first. We'll learn some techniques later in Chapters 6, 14 and 15 to deal with this. In the meantime we'll just leave Jonathan Harker with `{}` for `link lover`.

Back to the query: what if we just want to say `true` or `false` depending on if the character has a lover? To do that we can add a computed property to the query, using `EXISTS`. `EXISTS` will return `true` if a set is returned, and `false` if it gets `{}` (if there is nothing). This is once again a result of not having null in EdgeDB. It looks like this:

```edgeql
SELECT Person {
  name,
  is_single := NOT EXISTS Person.lover,
};
```

Now this prints:

```
{
  default::NPC {name: 'Jonathan Harker', is_single: true},
  default::NPC {name: 'The innkeeper', is_single: true},
  default::NPC {name: 'Mina Murray', is_single: false},
  default::PC {name: 'Emil Sinclair', is_single: true},
  default::Vampire {name: 'Count Dracula', is_single: true},
}
```

This also shows why abstract types are useful. Here we did a quick search on `Person` for data from `Vampire`, `PC` and `NPC`, because they all come from `abstract type Person`.

We've got a lot of single characters, let's try selecting just one of them:

```
SELECT Person {
  name,
  is_single := NOT EXISTS Person.lover,
} FILTER .is_single LIMIT 1;
```

This prints:
```
{default::Vampire {name: 'Count Dracula', is_single: true}}
```

Notice that we use use `LIMIT 1` instead of `assert_single()` because we want just one result, but we don't care that there are more that fit our `FILTER`. Using `assert_single()` would cause the database give us an error in case of multiple results. And of course `LIMIT 2`, `LIMIT 3` etc. will work just fine if we want to get a certain maximum number of objects.

It's possible that the query above doesn't actually select Dracula, but instead some other single `Person`. The `LIMIT 1` picks at most one result, but it says nothing about which one, just the first one that the database finds. Picking the order in which results get looked at is covered in [Chapter 10](../chapter10/index.md).

We can also put the computed property in the type itself. Here's the same computed property except now it's inside the `Person` type:

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  link lover -> Person;
  property is_single := NOT EXISTS .lover;
}
```

We won't keep `is_single` in the type definition though, because it's not useful enough for our game.

You might be curious about how computed links and properties are represented in databases on the back end. They are interesting because they {ref}`don't show up in the actual database <docs:ref_datamodel_computed>`, and only appear when you query them. And of course you don't specify the type because the computed expression itself determines the type. You can kind of imagine this when you look at a query with a quick computed variable like `SELECT country_name := 'Romania'`. Here, `country_name` is computed every time we do a query, and the type is determined to be a string. A computed link or property on a type does the same thing. But nevertheless, they still work in the same way as all other links and properties because the instructions for the computed ones are part of the type itself and do not change. In other words, they are a bit different on the back but the same up front.

## Ways to tell time

We will now learn about time, because it might be important for our game. Remember, vampires can only go outside at night.

The part of Romania where Jonathan Harker is has an average sunrise of around 7 am and a sunset of 7 pm. This changes by season, but to keep it simple we will just use 7 am and 7 pm to decide if it's day or night.

EdgeDB uses two major types for time.

- `std::datetime`, which is very precise and always has a timezone. Times in `datetime` use the ISO 8601 standard.
- `cal::local_datetime`, which doesn't worry about timezone.

There are two others that are almost the same as `cal::local_datetime`:

- `cal::local_time`, when you only need to know the time of day, and
- `cal::local_date`, when you only need to know the month, the day, and year.

We'll start with `cal::local_time` first.

`cal::local_time` is easy to create, because you can just cast to it from a `str` in the format 'HH:MM:SS':

```edgeql
SELECT <cal::local_time>('15:44:56');
```

This gives us the output:

```
{<cal::local_time>'15:44:56'}
```

We will imagine that our game has a clock that gives the time as a `str`, like the '15:44:56' in the example above. We'll make a quick `Time` type that can help. It looks like this:

```sdl
type Time { 
  required property clock -> str; 
  property clock_time := <cal::local_time>.clock; 
  property hour := .clock[0:2]; 
} 
```

`.clock[0:2]` is an example of {eql:op}`"slicing" <docs:arrayslice>`. `[0:2]` means start from index 0 (the first index) and stop _before_ index 2, which means indexes 0 and 1. This is fine because to cast a `str` to `cal::local_time` you need to write the hour with two numbers (e.g. 09 is okay, but 9 is not).

So this won't work:

```edgeql
SELECT <cal::local_time>'9:55:05';
```

It gives this error:

```
ERROR: InvalidValueError: invalid input syntax for type cal::local_time: '9:55:05'
  Hint: Please use ISO8601 format. Examples: 18:43:27 or 18:43 Alternatively "to_local_time" function provides custom formatting options.
```

Because of that, we are sure that slicing from index 0 to 2 will give us two numbers that indicate the hour of the day.

Now with this `Time` type, we can get the hour by doing this:

```edgeql
INSERT Time {
    clock := '09:55:05',
};
```

And then we can `SELECT` our `Time` objects and everything inside:

```edgeql
SELECT Time {
  clock,
  clock_time,
  hour,
};
```

That gives us a nice output that shows everything, including the hour:

`{default::Time {clock: '09:55:05', clock_time: <cal::local_time>'09:55:05', hour: '09'}}`.

Finally, we can add some logic to the `Time` type to see if vampires are awake or asleep. We could use an `enum` but to be simple, we will just make it a `str`.

```sdl
type Time {
  required property clock -> str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19
    ELSE 'awake';
}
```

So `awake` is calculated like this:

- First EdgeDB checks to see if the hour is greater than 7 and less than 19 (7 pm). But it's better to compare with a number than a string, so we write `<int16>.hour` instead of `.hour` so it can compare a number to a number.
- Then it gives us a string saying either 'asleep' or 'awake' depending on that.

Now if we `SELECT` this with all the properties, it will give us this:

`{default::Time {clock: '09:55:05', clock_time: <cal::local_time>'09:55:05', hour: '09', awake: 'asleep'}}`

One more note on `ELSE`: you can keep on using `ELSE` as many times as you like in the format `(result) IF (condition) ELSE`. Here's an example:

```
property awake := 'just waking up' IF <int16>.hour = 19 ELSE
                  'going to bed' IF <int16>.hour = 6 ELSE
                  'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE
                  'awake';
```

## SELECT while you INSERT

Back in Chapter 3, we learned how to select while deleting at the same time. You can do the same thing with `INSERT` by enclosing it in brackets and then selecting that, same as with any other `SELECT`. Because when we insert a new `Time`, all we get is a `uuid`:

```edgeql
INSERT Time {
  clock := '22:44:10'
};
```

The output is just something like this: `{default::Time {id: 3f6951c6-ff48-11eb-915e-3fd1092b2757}}`

So let's wrap the whole entry in `SELECT ()` so we can display its properties as we insert it. Because it's enclosed in brackets, EdgeDB will do that operation first, and then give it for us to select and do a normal query. Besides the properties to display, we can also add a computed property while we are at it. Let's give it a try:

```edgeql
SELECT ( # Start a selection
  INSERT Time { # Put the insert inside it
    clock := '22:44:10'
  }
) # The bracket finishes the selection
  { # Now just choose the properties we want
    clock,
    hour,
    awake,
    double_hour := <int16>.hour * 2
  };
```

Now the output is more meaningful to us: `{default::Time {clock: '22:44:10', hour: '22', awake: 'awake', double_hour: 44}}` We know the clock and the hour, we can see that vampires are awake, and even make a computed property from the object we just entered.

[Here is all our code so far up to Chapter 4.](code.md)

<!-- quiz-start -->

## Time to practice

1. This insert is not working.

   ```edgeql
   INSERT NPC {
     name := 'I Love Mina',
     lover := assert_single(
       (SELECT NPC FILTER .name LIKE '%Mina%')
     )
   };
   ```

   The error is: `invalid reference to default::NPC: self-referencing INSERTs are not allowed`. What keyword can we use to make this insert work?

   Bonus: there is another method we could use too to make it work without the keyword. Can you think of another way?

2. How would you display up to 2 `Person` types (and their `name` property) whose names include the letter `a`?

3. How would you display all the `Person` types (and their names) that have never visited anywhere?

   Hint: all the `Person` types for which `.places_visited` returns `{}`.

4. Imagine that you have the following `cal::local_time` type:

   ```edgeql
   SELECT has_nine_in_it := <cal::local_time>'09:09:09';
   ```

   This displays `{<cal::local_time>'09:09:09'}` but instead you want to display {true} if it has a 9 and {false} otherwise. How could you do that?

5. We are inserting a character called The Innkeeper's Son:

   ```edgeql
   INSERT NPC {
     name := "The Innkeeper's Son",
     age := 10
   };
   ```

   How would you `SELECT` this insert at the same time to display the `name`, `age`, and `age_ten_years_later` that is made from `age` plus 10?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan's curiosity gets the better of him. What's inside this castle?_
