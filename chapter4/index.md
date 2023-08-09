---
tags: Local Time, Advanced Filtering
leadImage: illustration_04.jpg
---

# Chapter 4 - "What a strange man this Count Dracula is."

The days and nights continue to go by, and Jonathan Harker is still in the castle. In this chapter it's time for us to learn how to work with time.

> Jonathan Harker wakes up late and is alone in the castle. Dracula appears after nightfall and they talk **through the night**. Dracula is making plans to move to London, and Jonathan gives him some advice about buying houses. Jonathan tells Dracula that a big house called Carfax would be a good house to buy. It's very big and quiet. Carfax is close to a mental asylum, but not too close. Dracula loves the idea.
>Dracula then tells Jonathan not to go into any of the locked rooms in the castle, because it could be dangerous. Jonathan sees that it's almost **morning**. They talked through the whole night again! Dracula suddenly stands up and says he must go, and leaves the room. Jonathan thinks about **Mina** back in London, who he is going to marry when he returns. He is beginning to feel that there is something wrong with Dracula, and the castle. Seriously, where are the other people?

First let's create Jonathan's girlfriend, Mina Murray. It would be nice to represent their relationship somehow, so let's try by adding a new link to the `Person` type in the schema called `lover`. Let's change the `Person` type to what you see here and do a migration:

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Place;
  lover: Person;
}
```

With this we can link the two of them together. We will assume that a person can only have one `lover`, so this is a single link. If we wanted `lover` to be a multi link, we would have written `multi lover` instead.

Mina is in London, and we don't know if she has been anywhere else. So let's do a quick insert to create the city of London. It couldn't be easier:

```edgeql
insert City {
  name := 'London',
};
```

To give her the city of London, we can just do a quick `(select City filter .name = 'London')`. This will give her the `City` that matches `.name = 'London'`, but it won't give an error if the city's not there: it will just return an empty set. When giving her Jonathan Harker as a `lover` link, however, it is a bit more complicated. Let's see why.

## Using the keywords detached, exists, and limit

The `insert` for Mina Murry is a bit more complicated. We might be tempted to try an `insert` this way, but there are two problems:

```edgeql
insert NPC {
  name := 'Mina Murray',
  lover := (select NPC filter .name = 'Jonathan Harker'),
  places_visited := (select City filter .name = 'London'),
 };
```

Fortunately, the compiler is smart enough to tell us exactly what the problems are! If you try this insert you will see the following error message:

```
error: QueryError: invalid reference to default::NPC: 
self-referencing INSERTs are not allowed
  ┌─ <query>:3:20
  │
3 │   lover := (select NPC filter .name = 'Jonathan Harker'),
  │                    ^^^ Use DETACHED if you meant to refer 
  to an uncorrelated default::NPC set
```

The issue here is that we are inside of an `insert` for the `NPC` type, but we want to link to another `NPC`, not the one we are inserting. We need to add `detached` to specify that we are talking about `NPC` in general, not the `NPC` that we are inserting right now.

Sounds good. Let's change `select NPC` to `select detached NPC` so that we can filter on all of the `NPC` objects. There is one error left, and you might be able to guess what it is. What will the result of the `filter` be, and does it match with the link `lover`?

```edgeql
insert NPC {
  name := 'Mina Murray',
  lover := (select detached NPC filter .name = 'Jonathan Harker'),
  places_visited := (select City filter .name = 'London'),
 };
```

That's right! Our `filter` will return a set of `NPC` objects. This set might have zero or one `NPC` objects...but it also might be more than one. That's not okay for a `link`, which is by default a `single link`. Here is the error message:

```
error: QueryError: possibly more than one element returned by an expression 
for a link 'lover' declared as 'single'
  ┌─ <query>:3:3
  │
3 │   lover := (select detached NPC filter .name = 'Jonathan Harker'),
  │   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ error
```

To fix this, we can use a function called {eql:func}`docs:std::assert_single`. This is used here because the link is a single link. EdgeDB doesn't know how many results we might get: for all it knows, there might be 2 or 3 or more `Jonathan Harkers`. To guarantee that we are only creating a single link, we use the `assert_single()` function. Careful! This will return an error if more than one result is returned.

The full insert for Mina Murray will look like this:

```edgeql
insert NPC {
  name := 'Mina Murray',
  lover := assert_single(
    (select detached NPC filter .name = 'Jonathan Harker')
  ),
  places_visited := (select City filter .name = 'London'),
};
```

EdgeDB has another keyword called `limit` that limits the number of objects returned to any number we like, and in this case we could have gotten the `insert` to work by using `limit 1`.

```edgeql
insert NPC {
  name := 'Mina Murray',
  lover := (select detached NPC filter .name = 'Jonathan Harker' limit 1),
  places_visited := (select City filter .name = 'London'),
 };
```

However, in this case `assert_single()` is better because `filter .name = 'Jonathan Harker'` might have returned more than one object. Maybe there are four or five Jonathan Harkers in the database, and only one of them would be returned. And if we used `limit 1` again in the same way, we might get a different `NPC` called Jonathan Harker. Using `limit 1` picks at most one result, but it says nothing about which one, just the first one that the database finds. Picking the order in which results get looked at is covered in [Chapter 10](../chapter10/index.md).

We can (and will) add a constraint to ensure that names are unique, but in the meantime keep in mind the difference between `assert_single()` and `limit 1`.

Now let's make a query to see who is single and who is not. This is easy by using a "computed" property, where we can create a new variable that we define with `:=`. First here is a basic query showing the names of each `Person` object's `lover`:

```edgeql
select Person {
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

We can see that Mina Murray has a lover but Jonathan Harker does not yet, because he was inserted first when Mina Murray didn't exist yet. We'll learn some techniques later in Chapters 6, 14 and 15 to deal with this. In the meantime we'll just leave Jonathan Harker with `{}` for the `lover` link.

Back to the query: what if we just want to say `true` or `false` depending on if the character has a lover? To do that we can put a computed property in the query, using `exists`. We'll call it `is_single`, but since it's not in the schema for the `Person` type we could call it anything we like here. The keyword `not exists` will return `false` if a set is returned, and `true` if it gets `{}` (if there is nothing). This is once again one of the nice things about using empty sets in EdgeDB instead of null. It looks like this:

```edgeql
select Person {
  name,
  is_single := not exists .lover,
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

This also shows why abstract types are useful. Here we did a quick search on `Person` which returned data from `Vampire`, `PC`, and `NPC` objects, because they all extend the `abstract type Person`.

We've got a lot of single characters, let's try selecting just one of them:

```edgeql
select Person {
  name,
  is_single := not exists .lover,
} filter .is_single limit 1;
```

If the first `Person` type returned from the database is Count Dracula, then we will see the following output:

```
{default::Vampire {name: 'Count Dracula', is_single: true}}
```

Now this time we did want to use `limit 1` instead of `assert_single()` because we want just up to one result, even if there are multiple objects that fit our `filter`. Using `assert_single()` would cause the database give us an error in case of multiple results. Similarly, `limit 2`, `limit 3` and any other number will work just fine if we only want a certain maximum number of objects.

We could also put the computed property in the type itself, so let's do that. Here's the same computed property except now it's inside the `Person` type:

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Place;
  lover: Person;
  property is_single := not exists .lover;
}
```

Notice that we have written `property is_single` this time instead of just `is_single`. With computed properties and links we need to give EdgeDB a little bit more help by letting it know whether the computed expression will result in a `property` or a `link`. So if you are working on your schema and the property or link uses a `:=` to make it computed, don't forget to choose between `property` or `link`!

You might be curious about how computed links and properties are represented in databases on the back end. They are interesting because they {ref}`don't show up in the actual database <docs:ref_datamodel_computed>`, and only appear when you query them. Computed links also don't specify the type because the expression itself determines the type. You can kind of imagine this when you look at a query with a quick computed variable like `select country_name := 'Romania'`. Here, `country_name` is computed every time we do a query, and the type is determined to be a string. A computed link or property on a type does the same thing. But nevertheless, they still work in the same way as all other links and properties because the instructions for the computed ones are part of the type itself and do not change. In other words, they are a bit different on the back but mostly the same up front.

## Ways to tell time

We will now learn about time, which is important for our game. Keeping track of time is important in general, but is especially important for us because vampires can only go outside at night.

The part of Romania that Jonathan Harker is visiting has an average sunrise of around 7 am and a sunset of 7 pm. This changes by season, but to keep it simple we will just use 7 am and 7 pm to decide if it's day or night.

EdgeDB uses two major types for time:

- `std::datetime`, which is the most precise because it includes a timezone. Times in `datetime` use the [ISO 8601 standard](https://en.wikipedia.org/w/index.php?title=ISO_8601&oldid=1154675086).
- `cal::local_datetime`, which includes the date and time but no information about the timezone.

There are two others that each include half of the information found in `cal::local_datetime`:

- `cal::local_time`, for when you only need to know the time,
- `cal::local_date`, for when you only need to know the month, the day, and year.

Our first concern is whether vampires are sleeping or not, so we'll start with `cal::local_time` first. Take a close look at the name: this is the first time we have come across something in a different module in the standard library (it's `cal::local_time`, not `std::local_time`). The name `cal` here stands for calendar.

`cal::local_time` is easy to create, because you can just cast to it from a `str` in the format 'HH:MM:SS':

```edgeql
select <cal::local_time>('15:44:56');
```

This gives us the output:

```
{<cal::local_time>'15:44:56'}
```

We will imagine that our game engine has a clock that sends the database the time as a `str`, like the '15:44:56' in the example above. We'll make a quick `Time` type that will hold this `str` and use it to make two computed properties. It looks like this:

```sdl
type Time { 
  required clock: str; 
  property clock_time := <cal::local_time>.clock; 
  property hour := .clock[0:2]; 
} 
```

`.clock[0:2]` is an example of {eql:op}`"slicing" <docs:arrayslice>` that we learned about in Chapter 2. To review, `[0:2]` means start from index 0 (the first index) and stop _before_ index 2, which means indexes 0 and 1. This is fine because to cast a `str` to `cal::local_time` you need to write the hour with two numbers (e.g. `09` is okay, but `9` is not).

So this won't work:

```edgeql
select <cal::local_time>'9:55:05';
```

It gives this error:

```
edgedb error: InvalidValueError: invalid input syntax
for type cal::local_time: '9:55:05'
Hint: Please use ISO8601 format. Examples: 18:43:27 or 18:43
Alternatively "to_local_time" function provides custom formatting options.
```

Because of that, we are sure that slicing from index 0 to 2 will give us two numbers that indicate the hour of the day.

Let's do a migration so we can insert a `Time` object. It's pretty easy:

```edgeql
insert Time {
    clock := '09:55:05',
};
```

And then we can `select` our `Time` objects and everything inside:

```edgeql
select Time {
  clock,
  clock_time,
  hour,
};
```

That gives us a nice output that shows everything, including the hour:

```
{default::Time {clock: '09:55:05', clock_time: <cal::local_time>'09:55:05', hour: '09'}}
```

Finally, we can add some logic to the `Time` type to see if vampires are awake or asleep. Since this property requires choosing between one of multiple choices, an enum seems like a good choice.

```sdl
scalar type SleepState extending enum <Asleep, Awake>;

type Time {
  required clock: str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
    else SleepState.Awake;
}
```

So `vampires_are` is calculated like this:

- First EdgeDB checks to see if the hour is greater than 7 and less than 19 (7 pm). But it's better to compare with a number than a string, so we cast into an `int16` with `<int16>.hour` instead of `.hour` so it can compare a number to a number.
- Then it chooses between the 'Asleep' or 'Awake' values of the enum depending on that.

Now let's do a `select` again with all the properties:

```edgeql
select Time {*};
```

We then get this output:

```
{
  default::Time {
    id: 0fb1d964-1989-11ee-915e-5b585f9c9744,
    clock: '09:55:05',
    clock_time: <cal::local_time>'09:55:05',
    hour: '09',
    vampires_are: Asleep,
  },
}
```

One more note on `else`: you can keep on using `else` as many times as you like in the format `(result) if (condition) else`. Here's an example showing how this could work if we had more values on the `SleepState` enum:

```sdl
property vampires_are := 
  SleepState.JustWakingUp if <int16>.hour = 19 else
  SleepState.GoingToBed if <int16>.hour = 6 else
  SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19 else
  SleepState.Awake;
```

## Selecting what you just inserted

Back in Chapter 3, we learned how to `select` while deleting at the same time. Using an `insert` also returns the object (or objects) in question, so we can select them too. You can do this by enclosing the output in parentheses and then giving that a shape, same as with any other `select`. Let's work on this one step at a time.

If we insert a new `Time`, all we get is a `uuid`:

```edgeql
insert Time {
  clock := '22:44:10'
};
```

The output is just something like this: `{default::Time {id: 3f6951c6-ff48-11eb-915e-3fd1092b2757}}`

Now let's wrap it inside a `select`:

```edgeql
select(
  insert Time {
    clock := '22:44:10'
  }
);
```

The output is still the same so far. But now that we are using `select`, we can follow it up with brackets to provide it a shape. Besides the regular properties to display, we can also add a computed property while we are at it. This computed property will give you a taste of what we will learn in the next chapter.

```edgeql
select (
  insert Time {
    clock := '22:44:10'
  }
)
  {
    clock,
    hour,
    vampires_are,
    time_until_just_before_midnight := <cal::local_time>'23:59:59'
                         - <cal::local_time>.clock
  };
```

(The computed `time_until_midnight` property used '23:59:59' as its input because the range of a `local_time` is 0:00:00 to 23:59:59. If we needed more accuracy we could have added a `<duration>'1 second'` to the expression. We will learn how to work with durations in the next chapter.)

Now the output is more meaningful to us: 

```
{
  default::Time {
    clock: '22:44:10',
    hour: '22',
    vampires_are: Awake,
    time_until_just_before_midnight: <cal::relative_duration>'PT1H15M49S',
  },
}
```

We know the clock and the hour, we can see that vampires are awake, and even make a computed property from the object we just entered. Looks like we have one hour, 15 minutes, and 50 seconds until midnight.

## Making Time global

Depending on how closely you've been following this chapter, you probably have about three or four `Time` objects in the database by now. That feels a bit odd, because this `Time` type is most useful for determining the current state of the game: the current clock time, if vampires are awake or not, and so on. So instead of multiple `Time` objects floating around in the database, we should have a single global `Time` object instead.

So first let's delete all the `Time` objects in the database with `delete Time;` and make a change to the schema by making a `global` type called `time`. In EdgeDB a global type can either be:

* A scalar type if it isn't computed,
* Any type if it is computed.

In our case we are going to make `time` a computed global that selects all the `Time` objects in the database, which in our case should never be more than one. So we can define it using the `assert_single()` function again. The global type will now look like this:

```sdl
global time := assert_single((select Time));
```

Now that this is added, let's do a migration and see what's inside! Selecting a global type is done in the same way as with other types except that it needs the `global` keyword:

```edgeql
select global time {*};
```

Doing this should return an empty set. Now let's insert the current time:

```edgeql
insert Time { clock := '09:00:00' };
```

And then do the query on `global time` again:

```edgeql
select global time {*};
```

With this single global `Time` object in our database, we can now see the current time and what vampires are up to at the moment:

```
{
  default::Time {
    id: d6363f92-1517-11ee-ab3d-834f250e5e21,
    clock: '09:00:00',
    clock_time: <cal::local_time>'09:00:00',
    hour: '09',
    vampires_are: Asleep,
  },
}
```

In Chapter 6 we will learn how to `update` objects, which will let us change this global `Time` object whenever we like. And much later on in the book we will also add a global scalar value, so stay tuned!

[Here is all our code so far up to Chapter 4.](code.md)

<!-- quiz-start -->

## Time to practice

1. This insert is not working.

   ```edgeql
   insert NPC {
     name := 'I Love Mina',
     lover := assert_single(
       (select NPC filter .name like '%Mina%')
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
   select has_nine_in_it := <cal::local_time>'09:09:09';
   ```

   This displays `{<cal::local_time>'09:09:09'}` but instead you want to display {true} if it has a 9 and {false} otherwise. How could you do that?

5. We are inserting a character called The Innkeeper's Son:

   ```edgeql
   insert NPC {
     name := "The Innkeeper's Son",
     age := 10
   };
   ```

   How would you `select` this insert at the same time to display the `name`, `age`, and `age_ten_years_later` that is made from `age` plus 10?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan's curiosity gets the better of him. What's inside this castle?_
