# Chapter 4 - "What a strange man this Count Dracula is."

> Jonathan Harker wakes up late and is alone in the castle. Dracula appears after nightfall and they talk **through the night**. Dracula is making plans to move to London, and Jonathan gives him some advice about buying houses. Jonathan tells Dracula that a big house called Carfax would be a good house to buy. It's very big and quiet. It's close to an asylum for crazy people, but not too close. Dracula likes the idea. He then tells Jonathan not to go into any of the locked rooms in the castle, because it could be dangerous. Jonathan sees that it's almost morning - they talked through the whole night again. Dracula suddenly stands up and says he must go, and leaves the room. Jonathan thinks about **Mina** back in London, who he is going to marry when he returns. He is beginning to feel that there is something wrong with Dracula, and the castle. Seriously, where are the other people?

First let's create Jonathan's girlfriend, Mina Murray. But we'll also add a new link to the `Person` type in the schema called `lover`:

```
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  link lover -> Person;
}
```

With this we can link the two of them together. We will assume that a person can only have one `lover`, so this is a `single link` but we can just write `link`.

Mina is in London, and we don't know if she has been anywhere else. So let's do a quick insert to create the city of London. It couldn't be easier:

```
INSERT City {
    name := 'London',
};
```

To give her the city of London, we can just do a quick `SELECT City FILTER .name = 'London'`. This will give her the `City` that matches `.name = 'London'`, but it won't give an error if the city's not there: it will just return a `{}` empty set.

## DETACHED, LIMIT, and EXISTS

For `lover` it is the same process but a bit more complicated:

```
INSERT NPC {
  name := 'Mina Murray',
  lover := (SELECT DETACHED NPC Filter .name = 'Jonathan Harker' LIMIT 1),
  places_visited := (SELECT City FILTER .name = 'London'),
};
```

You'll notice two things here:

- `DETACHED`. This is because we are inside of an `INSERT` for the `NPC` type, but we want to link to the same type: another `NPC`. We need to add `DETACHED` to specify that we are talking about `NPC` in general, not the `NPC` that we are inserting right now.
- `LIMIT 1`. This is because the link is a `single link`. EdgeDB doesn't know how many results we might get: for all it knows, there might be 2 or 3 or more `Jonathan Harkers`. To guarantee that we are only creating a `single link`, we use `LIMIT 1`. And of course `LIMIT 2`, `LIMIT 3` etc. will work just fine if we want to link to a certain maximum number of objects.

Now we want to make a query to see who is single and who is not. This is easy by using a "computable", where we can create a new variable that we define with `:=`. First here is a normal query:

```
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
  Object {name: 'Jonathan Harker', lover: {}},
  Object {name: 'The innkeeper', lover: {}},
  Object {name: 'Mina Murray', lover: Object {name: 'Jonathan Harker'}},
  Object {name: 'Count Dracula', lover: {}},
}
```

Okay, so Mina Murray has a lover but Jonathan Harker does not yet, because he was inserted first. We'll learn some techniques later in Chapters 6, 14 and 15 to deal with this. In the meantime we'll just leave Jonathan Harker with `{}` for `link lover`.

Back to the query: what if we just want to say `true` or `false` depending on if the character has a lover? To do that we can add a computable to the query, using `EXISTS`. `EXISTS` will return `true` if a set is returned, and `false` if it gets `{}` (if there is nothing). This is once again a result of not having null in EdgeDB. It looks like this:

```
select Person {
  name,
  is_single := NOT EXISTS Person.lover,
};
```

Now this prints:

```
  Object {name: 'Count Dracula', is_single: true},
  Object {name: 'The innkeeper', is_single: true},
  Object {name: 'Mina Murray', is_single: false},
  Object {name: 'Jonathan Harker', is_single: true},
  Object {name: 'Emil Sinclair', is_single: true},
```

This also shows why abstract types are useful. Here we did a quick search on `Person` for data from `Vampire`, `PC` and `NPC`, because they all come from `abstract type Person`.

We can also put the computable in the type itself. Here's the same computable except now it's inside the `Person` type:

```
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  property lover -> Person;
  property is_single := not exists .lover;
}
```

We won't keep `is_single` in the type definition though, because it's not useful enough for our game.

You might be curious about how computables are represented in databases on the back end. They are interesting because they [don't show up in the actual database](https://www.edgedb.com/docs/datamodel/computables), and only appear when you query them. And of course you don't specify the type because the computable itself determines the type. You can kind of imagine this when you look at a query with a quick computable like `SELECT country_name := 'Romania'`. Here, `country_name` is computed every time we do a query, and the type is determined to be a string. A computable on a type does the same thing. But nevertheless, they still work in the same way as all other links and properties because the instructions for the computable are part of the type itself and do not change. In other words, they are a bit different on the back but the same up front.

## Ways to tell time

We will now learn about time, because it might be important for our game. Remember, vampires can only go outside at night.

The part of Romania where Jonathan Harker is has an average sunrise of around 7 am and a sunset of 7 pm. This changes by season, but to keep it simple we will just use 7 am and 7 pm to decide if it's day or night.

EdgeDB uses two major types for time.

-`std::datetime`, which is very precise and always has a timezone. Times in `datetime` use the ISO 8601 standard. -`cal::local_datetime`, which doesn't worry about timezone.

There are two others that are almost the same as `cal::local_datetime`:

-`cal::local_time`, when you only need to know the time of day, and -`cal::local_date`, when you only need to know the month and the day.

We'll start with `cal::local_time` first.

`cal::local_time` is easy to create, because you can just cast to it from a `str` in the format 'HH:MM:SS':

```
SELECT <cal::local_time>('15:44:56');
```

This gives us the output:

```
{<cal::local_time>'15:44:56'}
```

We will imagine that our game has a clock that gives the time as a `str`, like the '15:44:56' in the example above. We'll make a quick `Date` type that can help. It looks like this:

```
type Date {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
}
```

`.date[0:2]` is an example of ["slicing"](https://www.edgedb.com/docs/edgeql/funcops/array#operator::ARRAYSLICE). [0:2] means start from index 0 (the first index) and stop _before_ index 2, which means indexes 0 and 1. This is fine because to cast a `str` to `cal::local_time` you need to write the hour with two numbers (e.g. 09 is okay, but 9 is not).

So this won't work:

```
SELECT <cal::local_time>'9:55:05';
```

It gives this error:

```
ERROR: InvalidValueError: invalid input syntax for type cal::local_time: '9:55:05'
```

Because of that, we are sure that slicing from index 0 to 2 will give us two numbers that indicate the hour of the day.

Now with this `Date` type, we can get the hour by doing this:

```
INSERT Date {
    date := '09:55:05',
};
```

And then we can `SELECT` our `Date` types and everything inside:

```
SELECT Date {
  date,
  local_time,
  hour,
};
```

That gives us a nice output that shows everything, including the hour:

`{Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09'}}`.

Finally, we can add some logic to the `Date` type to see if vampires are awake or asleep. We could use an `enum` but to be simple, we will just make it a `str`.

```
type Date {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

So `awake` is calculated like this:

- First EdgeDB checks to see if the hour is greater than 7 and less than 19 (7 pm). But it's better to compare with a number than a string, so we write `<int16>.hour` instead of `.hour` so it can compare a number to a number.
- Then it gives us a string saying either 'asleep' or 'awake' depending on that.

Now if we `SELECT` this with all the properties, it will give us this:

`Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09', awake: 'asleep'}`

## SELECT while you INSERT

Back in Chapter 3, we learned how to select while deleting at the same time. You can do the same thing with `INSERT` by enclosing it in brackets and then selecting that, same as with any other `SELECT`. Because when we insert a new `Date`, all we get is a `uuid`:

```
INSERT Date {
  date := '22.44.10'
};
```

The output is just something like this: `{Object {id: 528941b8-f638-11ea-acc7-2fbb84b361f8}}`

So let's wrap the whole entry in `SELECT ()` so we can display its properties as we insert it. Because it's enclosed in brackets, EdgeDB will do that operation first, and then give it for us to select and do a normal query. Besides the properties to display, we can also add a computable while we are at it. Let's give it a try:

```
SELECT ( # Start a selection
  INSERT Date { # Put the insert inside it
    date := '22.44.10'
}) # The bracket finishes the selection
  { # Now just choose the properties we want
    date,
    hour,
    awake,
    double_hour := <int16>.hour * 2
  };
```

Now the output is more meaningful to us: `{Object {date: '22.44.10', hour: '22', awake: 'awake', double_hour: 44}}` We know the date and the hour, we can see that vampires are awake, and even make a computable from the object we just entered.

[Here is all our code so far up to Chapter 4.](code.md)

## Time to practice

<!-- quiz-start -->

1. This insert is not working.

   ```
   INSERT NPC {
     name := 'I Love Mina',
     lover := (SELECT NPC FILTER .name LIKE '%Mina%' LIMIT 1)
   };
   ```

   The error is: `invalid reference to default::NPC: self-referencing INSERTs are not allowed`. What keyword can we use to make this insert work?

   Bonus: there is another method we could use too to make it work without the keyword. Can you think of another way?

2. How would you display up to 2 `Person` types (and their `name` property) whose names include the letter `a`?

3. How would you display all the `Person` types (and their names) that have never visited anywhere?

   Hint: all the `Person` types for which `.places_visited` returns `{}`.

4. Imagine that you have the following `cal::local_time` type:

   ```
   SELECT has_nine_in_it := <cal::local_time>'09:09:09';
   ```

   This displays `{<cal::local_time>'09:09:09'}` but instead you want to display {true} if it has a 9 and {false} otherwise. How could you do that?

5. We are inserting a character called The Innkeeper's Son:

   ```
   INSERT NPC {
     name := "The Innkeeper's Son",
     age := 10
   };
   ```

   How would you `SELECT` this insert at the same time to display the `name`, `age`, and `age_ten_years_later` that is made from `age` plus 10?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 5: [Jonathan decides to explore the castle a bit. Just to be safe...](../chapter5/index.md)
