# Welcome!

Welcome to the tutorial to learn EdgeDB in plain English. "Plain English" means using simple phrases and short sentences so you can quickly get the information you need. This lets you learn EdgeDB with a minimum of distractions, and is helpful if English is your second (or third, fourth...) language.

Nevertheless, simple English doesn't mean boring. We will imagine that we are creating the database for a game that is based on the book Bram Stoker's Dracula. Because it will use the events in the book for the background, we need a database to tie everything together. It will need to show the connections between the characters, locations, dates, and more.


# Chapter 1 - Jonathan Harker travels to Transylvania

![thththt](Sample_image.png)

In the beginning of the book we see the main character Jonathan Harker, a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan still doesn't know that Count Dracula is a vampire, so he's enjoying the trip to a new part of Europe. Here is how the book begins, with parts that are good for a database in **bold**:

>**3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

## Schema, object types

This is already a lot of information, and it helps us start to think about our database schema (structure). EdgeQL uses something called [SDL (schema definition language)](https://edgedb.com/docs/edgeql/sdl/index#ref-eql-sdl) that makes migration easy. So far our schema needs the following:

- Some kind of City or Location type. Cities should have a name and a location, and sometimes a different name or spelling. Bistritz for example is now called Bistrița (it's in Romania), and Buda-Pesth is now written Budapest.
- Some kind of Person type. We need it to have a name, and also a way to track the places that the person visited.
 
To make a type in EdgeQL, just use the keyword `type` followed by the type name, then `{}` curly brackets. Our `Person` type will start out like this:

```
type Person {
}
```

Inside this we insert the properties for our `Person` type. Use `required property` if the type needs it, and just `property` if it is optional. 

```
type Person {
  required property name -> str;
  property places_visited -> array<str>;
}
```

A `str` is just a string, and goes inside either single quotes: `'Jonathan Harker'` or double quotes: `"Jonathan Harker"`. An `array` is a collection type, and our array here is an array of `str`s. We want it to look like this: `["Bistritz", "Vienna", "Buda-Pesth"]`. With that we can easily search later and see which character has visited where.

`places_visited` is not a `required` property because we might later add people that we don't care about. Maybe one person will be the "innkeeper_in_bistritz" or something, and we won't care about `places_visited` for him.

Now for our City type:

```
type City {
  required property name -> str;
  property modern_name -> str;
}
```

This is similar: all cities have a name in the book, but some don't have a different modern name. Vienna is still Vienna, for example. We are imagining that our game will link the city names to their modern names so we can easily place them on a map.

## Migration

We haven't created our database yet, though. There are two small steps that we need to do first [after installing EdgeDB](https://edgedb.com/download). First we create a database with the `CREATE DATABASE` keyword and our name for it:

```
CREATE DATABASE dracula;
```

Then we type ```\c dracula``` to connect to it.

Lastly, we we need to do a migration. This will give the database the structure we need to start interacting with it. Migrations are not difficult with EdgeDB:

- First you start them with `START MIGRATION TO {}`
- Inside this you add at least one `module`, so your types can be accessed. A module is a namespace, a place where similar types go together. If you wrote `module default` for example and then `type Person`, the type `Person` would be at `default::Person`. And in EdgeQL, `std::bytes` for example is the type `bytes` in the standard library (std).
- Then you add the types we mentioned above, and type `POPULATE MIGRATION` to add the data.
- Finally, you type `COMMIT MIGRATION` and the migration is done.

## Inserting objects

Later on we can think about adding time zones and locations for the cities for our imaginary game. But in the meantime, we will add some items to the database using `INSERT`. By the way, keywords in EdgeDB are case insensitive, so `INSERT`, `insert` and `InSeRT` are all the same. But using capital letters is the normal practice for databases so it's best to use that.

We use the `:=` operator to declare, while `=` is used for equality (not `==`).

```
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Vienna", "Buda-Pesth"],
};

INSERT City {
  name := 'Munich',
};

INSERT City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};
```

As you can see, `str`s are fine with unicode letters like ț. Even emojis are just fine: you could create a `City` called '🤠' if you wanted to.

Every time you `INSERT` an item, EdgeDB gives you a `uuid` back. That's the unique number for the item. It will look like this:

```
{Object {id: d2af670c-f1d6-11ea-a30f-8b40bc5413e0}}
```

It is also what shows up when you use `SELECT` to select a type. `SELECT` is the basic query command in EdgeDB that will give you a result based on the input that comes after `SELECT`. Just typing `SELECT` with a type will show you all the `uuid`s for the type. Let's look at all the cities we have so far:

```
SELECT City;
```

This gives us three items:

```
{
  Object {id: d2b64e00-f1d6-11ea-a30f-1f161d0b15ae},
  Object {id: d2c023b2-f1d6-11ea-a30f-e3069a47b57e},
  Object {id: d37bc838-f1d6-11ea-a30f-afb031317264},
}
```

Now let's add a property to the query. We'll `SELECT` all `City` types with `modern_name`. Here is our query:

```
SELECT City {
  modern_name,
};
```

You will remember that one of our cities doesn't have a modern name. It still shows up though as an "empty set", because every value in EdgeDB is a set of elements. Here is the result:

```
{Object {modern_name: {}}, Object {modern_name: 'Budapest'}, Object {modern_name: 'Bistrița'}}
```

So there is some object with an empty set for `modern_name`, while the other two have a name. This shows us that EdgeDB doesn't have `null` like in some languages: if nothing is there, it will return an empty set.

If you just want to return the names without the object structure, you can write `SELECT City.modern_name`. That will give this output:

```
{'Budapest', 'Bistrița'}
```

You can also change names like `modern_name` to something else if you want by using `:=` after the name you want. For example:

```
SELECT City {
  name_in_dracula := .name,
  name_today := .modern_name,
};
```

This prints:

```
{
  Object {name_in_dracula: 'Munich', name_today: {}},
  Object {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest'},
  Object {name_in_dracula: 'Bistritz', name_today: 'Bistrița'},
}
```

## Links

The last thing we should do is change the properties for `Person`. Right now, `places_visited` gives us the names we want, but it makes more sense to link `Person` and `City` together. We'll change `Person` to this:

```
type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
}
```

We wrote `MULTI` in front of `LINK` because one `Person` should be able to link to more than one `City`. The opposite of `MULTI` is `SINGLE`, but if you just write `LINK` then EdgeDB will treat it as `SINGLE`.

Now when we `INSERT` Jonathan Harker, he will be connected to the type `City`. Technically, we can now just enter this to create him now:

```
INSERT Person {
  name := 'Jonathan Harker',
};
```

But this will only create a `Person` type connected to the `City` type with nothing in it. Let's see what's inside:

```
SELECT Person {
  name,
  places_visited
};
```

Here is the output: `{Object {name: 'Jonathan Harker', places_visited: {}}}`

That's not good enough. We'll change `places_visited` when we `INSERT` to `places_visited := City`. Now it will put all the `City` types in there. Now let's see the places that Jonathan has visited. The code below is almost but not quite what we need: 

```
select Person {
  name,
  places_visited
};
```

Here is the output:

```
  Object {
    name: 'Jonathan Harker',
    places_visited: {
      Object {id: 5ba2c9b2-f64d-11ea-acc7-d3a96742e345},
      Object {id: 5bbb2368-f64d-11ea-acc7-ffb002eabe1a},
      Object {id: 5bc2bba0-f64d-11ea-acc7-8b468bbfae39},
    },
  },
```

Close! Now we just need to let EdgeDB know that we want to display the `name` property of the `City` type. To do that, we add a colon and then put `name` inside curly brackets - same as with any other `SELECT`.

```
select Person {
  name,
  places_visited: {
    name
  }
};
```

Success! Now we get the output we wanted:

```
  Object {
    name: 'Jonathan Harker',
    places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}},
  },
```

Right now we only have three `City` objects, so this is no problem yet. But later on we will have more cities and we will have to use `FILTER`. We will learn that in the next chapter.

[Here is all our code so far up to Chapter 1.](chapter_1_code.md)

# Chapter 2 - At the Hotel in Bistritz

We continue to read the story as we think about the database we need to store the information. The important information is in bold:

>Jonathan Harker has found a hotel in **Bistritz**, called the **Golden Krone Hotel**. He gets a welcome letter there from Dracula, who is waiting in his **castle**. Jonathan Harker will have to take a **horse-driven carriage** to get there tomorrow. We also see that Jonathan Harker is from **London**. The innkeeper at the Golden Krone Hotel seems very afraid of Dracula. He doesn't want Jonathan to leave and says it will be dangerous, but Jonathan doesn't listen.

## DDL

Right away we see that we could add another property to `City`, and will write this: `property important_places -> array<str>;` It will now look like this:

```
type City {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
```

So we can add it by using SDL when we do the migration. On top of SDL, EdgeQL also has something called [DDL (data definition language)](https://edgedb.com/docs/edgeql/ddl/index) that is more low level. DDL is good for quick changes, but not as good for large migrations. One reason why is that the order matters in DDL, but not in SDL. That means that if you have a type `City` that is connected to a type `Country`, you need to create `Country` first in DDL. In SDL, the other doesn't matter. 

But here we will look at a quick example of DDL. If we've already started our database, we can quickly change it like this:

```
ALTER TYPE City {
  CREATE PROPERTY important places -> array<str>
};
```

And it will say `OK: ALTER` to tell us that it worked. For most of this course we will keep using SDL, so when we change a type we are changing the migration structure. But you can remember that DDL is also an option for quick changes when you need them.

## Scalar types, extending

We now have two types of transport in the book: train, and horse-drawn carriage. This book is based in the late 1800s and our game will let the characters use different types of transport too. Here an `enum` is probably the best choice, because an `enum` is about making one choice between options. Here we see the word `scalar` for the first time: this will be a `scalar type` because it only holds a single value at a time. The other types (`City`, `Person`) are `object types` because they hold any number of values at the same time.

The other keyword we will see for the first time is `extending`. This gives you all the power of the type that you are extending. We will write our `Tranport` type like this:

```
scalar type Transport extending enum<'feet', 'train', 'horse-drawn carriage'>;
```

Did you notice that `scalar type` ends with a semicolon and the other types don't? That's because the other types have a `{}` to make a full expression. But here on a single line we don't have `{}` so we need the semicolon to show that the expression ends here.

This `Transport` type, however, is going to be useful for the characters in our game - not the characters in the book. Only characters in our game will be choosing between one type of transport or another. So maybe it's a good time to turn our `Person` type into something else. What we can do is make `Person` an `abstract type` instead. An `abstract type` is a sort of base type that you can use to build other types. Then we can have two other types called `PC` and `NPC` that draw from it with the keyword `extending`.

So now this part of the schema looks like this:

```
abstract type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
}
type PC extending Person {
  required property transport -> Transport;
}
type NPC extending Person {
}
```

Now the characters from the book will be `NPC`s (non-player characters), while `PC` is being made with our game in mind. Because `Person` is now an abstract type, we can't use it directly anymore. It will give us this error if we try:

```
error: cannot insert into abstract object type 'default::Person'
  ┌─ query:1:8
  │
1 │ INSERT Person {
  │        ^^^^^^^ error
```

No problem - just change `Person` to `NPC` and it will work.

Let's also experiment with a player character. We'll make one called Emil Sinclair who starts out traveling by horse-drawn carriage. We'll also just give him `City` so he'll have all three cities.

```
INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := <Transport>'horse-drawn carriage',
};
```

Note that we didn't just write `'horse-drawn carriage'`, because transport is an `enum<str>`, not a `str`. The `<>` angle brackets do casting, meaning to change one type into another. EdgeDB won't try to change one type into another unless you ask it to with casting. That's why this won't give us `true`:

```
SELECT 'feet' IS Transport;
```

We will get an output of `{false}`, because 'feet' is just a `str` and nothing else. But this will work:

```
SELECT <Transport>'feet' IS Transport;
```

Then we get `{true}`.

You can cast more than once at a time if you need to. This example isn't something you will ever do but shows how you can cast over and over again if you want:

```
SELECT <str><int64><str><int32>50 is str; 
```

That also gives us `{true}` because all we did is ask if it is a `str`, which it is. 

Casting works from right to left, with the final cast on the far left. So `<str><int64><str><int32>50` means "50 into an int32 into a string into an int64 into a string". Or you can read it left to right like this: "A string from an int64 from a string from an int32 from the number 50".

## Filter

Finally, let's learn how to `FILTER` before we're done Chapter 2. You can use `FILTER` after the curly brackets in `SELECT` to only show certain results. Let's `FILTER` to only show `Person` types that have the name 'Emil Sinclair':

```
select Person {
  name,
  places_visited: {name},
} FILTER .name = 'Emil Sinclair';
```

`FILTER .name` is short for `FILTER Person.name`. You can write `FILTER Person.name` too if you want - it's the same thing.

The output is this: 

```
{Object {name: 'Emil Sinclair', places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}}}}
```

Let's filter the cities now. You can use `LIKE` or `ILIKE` to match on parts of a string. `LIKE` is case-sensitive ("Bistritz" matches "Bistritz" but "bistritz" does not), while `ILIKE` is not. You can also add `%` on the left and/or right which means match anything. So `ILIKE '%IsTRiT%'` would match Bistritz.

Let's `FILTER` to get all the cities that start with a capital B. We'll use `LIKE` because it's case-sensitive:

```
SELECT City {
  name,
  modern_name,
} FILTER .name LIKE 'B%';
```

Here is the result:

```
  Object {name: 'Buda-Pesth', modern_name: 'Budapest'},
  Object {name: 'Bistritz', modern_name: 'Bistrița'},
```

You can also index a string so one other way to do it is like this:

```
SELECT City {
  name,
  modern_name,
} FILTER .name[0]; = 'B'; # First character must be 'B'
```

That gives the same result. Careful though: if you set the number too high then it will try to search outside of the string, which is an error. If we change 0 to 18, we'll get this:

```
ERROR: InvalidValueError: string index 18 is out of bounds
```

And if you have any `City` types with a name of `''`, even a search for index 0 will cause an error. But if you use LIKE or ILIKE with an empty parameter, it will just give an empty set: `{}` instead of an error.

[Here is all our code so far up to Chapter 2.](chapter_2_code.md)

# Chapter 3 - Jonathan goes to Castle Dracula

In this chapter we are going to start to think about time, as you can see from what Jonathan Harker is doing:

>Jonathan Harker has just arrived at Castle Dracula after a ride in the carriage through the mountains. The ride was terrible: there was snow, strange blue fires and wolves everywhere. It was night when he arrived, and he meets and talks with Count Dracula. Dracula leaves before the sun rises though, because vampires are hurt by sunlight. Jonathan doesn't know that he's a vampire yet.

This is a good time to create a `Vampire` type. We can extend it from `abstract type Person` because that type only has `name` and `places_visited`, which are good for `Vampire` too. But vampires are different from humans because they can live forever. Let's add `age` to `Person` so that all the other types can use it too. Now `Person' looks like this:

```
abstract type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
  property age -> int16;
}
```

`int16` means a 16 bit (2 byte) integer, which has enough space for -32768 to +32767. That's enough for age, so we don't need the bigger `int32` or `int64` types which are much larger. We also don't want it a `required property`, because we won't know everybody's age.

First we'll make `Vampire` a type that extends `Person`, and adds age:

```
type Vampire extending Person {            
  property age -> int16;
}
```

We will also take `age` out of `Person`, because `Vampire` specifies it. 

Now we can create Count Dracula. We know that he lives in Romania, but that isn't a city. This is a good time to change the `City` type. We'll change the name to `Place` and make it an `abstract type`, and then `City` can extend from it. We'll also add a `Country` type that does the same thing. Now they look like this:

```
abstract type Place {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
type City extending Place;
type Country extending Place;
```

Now it's easy to make a `Country`, just do an insert and give it a name. We'll quickly insert a `Country` objects with the name Hungary.

We are now ready to make Dracula. Now, `places_visited` is still defined as a `Place`, and that includes lots of things: London, Bistritz, Hungary, etc. We only know that Dracula has been in Romania, so we can do a quick `FILTER` instead, inside `()`:

```
insert Vampire {
  name := 'Count Dracula',
  places_visited := (SELECT Place FILTER .name = 'Romania'),
};
{Object {id: 0a1b83dc-f2aa-11ea-9f40-038d228e2bba}}
```

The `uuid` there is the reply from the server showing that we were successful.

Let's check if `places_visited` worked. We only have one `Vampire` object now, so let's `SELECT` it:

```
SELECT Vampire {
  places_visited: {
    name
  }
};
```

This gives us: `{Object {places_visited: {Object {name: 'Romania'}}}}` Perfect.

Now let's think about `age`. It was easy for the `Vampire` type, because they can live forever. But now we want to give `age` to the `PC` and `NPC` types too. But we don't want them to live up to 32767 years, because they aren't the type `Vampire`. For this we can add a "constraint". Instead of `age`, we'll give them a new type called `HumanAge`. Then we can write `constraint` and use [one of the functions](https://edgedb.com/docs/datamodel/constraints) that it can use. We will use `max_value()`. Now it looks like this:

```
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

Then add it to the `NPC` type:

```
type NPC extending Person {
  property age -> HumanAge;
}
```

Now it won't be able to be more than 120. So if we write this:

```
insert NPC {
    name := 'The innkeeper',
    age := 130
};
```

It won't work. Here is the error: `ERROR: ConstraintViolationError: Maximum allowed value for HumanAge is 120.` Perfect.

Now if we change `age` to 30, we get a message showing that it worked: `{Object {id: 72884afc-f2b1-11ea-9f40-97b378dbf5f8}}`. Now no NPCs can be over 120 years old.

[Here is all our code so far up to Chapter 3.](chapter_3_code.md)

# Chapter 4 - "What a strange man this Count Dracula is."

>Jonathan Harker wakes up late and is alone in the castle. Dracula appears after nightfall and they talk **through the night**. Dracula is making plans to move to London, and Jonathan gives him some advice. Dracula tells him not to go into any of the locked rooms, because it could be dangerous. Then he quickly leaves when he sees that it is almost morning. Jonathan thinks about **Mina** back in London, who he is going to marry when he returns. He is beginning to feel that there is something wrong with Dracula, and the castle. Where are the other people?

First let's create Jonathan's girlfriend, Mina Murray. But we'll also add a new link to the `Person` type in the schema:

```
abstract type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
  LINK lover -> Person;
}
```

We will assume that a person can only have one `lover`, so this is a `SINGLE LINK` but we can just write `LINK`.

Mina is in London, and we don't know if she has been anywhere else. So for the meantime, we are just going to create the city of London. It couldn't be easier:

```
INSERT City {
    name := 'London',
};
```

To give her the city of London, we will just do a quick `SELECT City FILTER .name = 'London'`. For `lover` it is the same process but a bit mork complicated:

```
INSERT NPC {
  name := 'Mina Murray',
  lover := (SELECT DETACHED NPC Filter .name = 'Jonathan Harker' LIMIT 1),
  places_visited := (SELECT City FILTER .name = 'London'),
};
```

You'll notice two things here:

- `DETACHED`. This is because we are inside of an `INSERT` for the `NPC` type. We need to add `DETACHED` to tell EdgeDB that we are talking about a different `NPC`, not self.
- `LIMIT 1`. This is because the link is a `SINGLE LINK`. The search for 'Jonathan Harker' might give us more than one person with this name, so we need to make sure we only get one.

We will also add Mina to Jonathan Harker as well in the same way. Now we want to make a query to see who is single and who is not. This is easy by using a "computable", something that lets us create a new variable that we define with `:=`. First here is a normal query:

```
select Person {
  name,
  lover: {
    name
  }
};
```

This gives us:

```
  Object {name: 'Count Dracula', lover: {}},
  Object {name: 'Mina Murray', lover: 'Jonathan Harker'},
  Object {name: 'Jonathan Harker', lover: 'Mina Murray'},
```

But we don't want to have to read every line ourselves - we just want `true` or `false`. Now we'll add the computable to the query:

```
select Person {
  name,
  is_single := NOT EXISTS Person.lover,
};
```

Now this prints:

```
  Object {name: 'Count Dracula', is_single: true},
  Object {name: 'Mina Murray', is_single: false},
  Object {name: 'Jonathan Harker', is_single: false},
```

This also shows why abstract types are useful. Here we did a quick search on `Person` for data from both `Vampire` and `NPC`, because they both come from `abstract type Person`.

We can also put a computable in the type itself.

```
abstract type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
  property lover -> Person;
  property is_single := NOT EXISTS .lover;
}
```

We won't keep `is_single` in the type definition though, because it's not useful enough for our game.

We will now learn about time, because it might be important for our game: vampires can only go outside at night.

The part of Romania where Jonathan Harker is has an average sunrise of around 7 am and a sunset of 7 pm. To keep it simple, we will use that to decide if it's day or night.

EdgeDB uses two types for time: 

-`std::datetime`, which is very precise and always has a timezone. Times in `datetime` use the ISO 8601 standard.
-`cal::local_time`, which doesn't worry about the timezone. We will use this first.

`cal::local_time` is easy to create, because you can just cast to it from a `str`:

```
SELECT <cal::local_time>('15:44:56');
```

This gives us the output:

```
{<cal::local_time>'15:44:56'}
```

We will imagine that our game has a clock that gives the time as a `str`. We'll make a quick `Date` type that can help. It looks like this:

```
type Date {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
}
```

`.date[0:2]` is an example of "slicing". [0:2] means start from index 0 (the first index) and stop *before* index 2, which means indexes 0 and 1. This is fine because to cast a `str` to `cal::local_time` you need to write the hour with two numbers (e.g. 09 instead of 9).

So this won't work:

```
SELECT <cal::local_time>'9:55:05';
```

It gives this error:

```
ERROR: InvalidValueError: invalid input syntax for type cal::local_time: '9:55:05'
```

So with this `Date` type, we can get the hour by doing this:

```
INSERT Date {
    date := '09:55:05',
};
```

And then we can `SELECT` our `Date` types:

```
SELECT Date {
  date,
  local_time,
  hour,
};
```

That gives us a nice output that shows everything, including the hour: `{Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09'}}`.

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

- First EdgeDB checks to see if the hour is greater than 7 and less than 19 (7 pm). But it can't compare on a `str`, so we write `<int16>.hour` instead of `.hour` so it can compare a number to a number.
- Then it gives us a string saying either 'asleep' or 'awake' depending on that.

Now if we `SELECT` this with all the properties, it will give us this: `  Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09', awake: 'asleep'}`

One good trick to know is that you can `SELECT` the item you just `INSERT`ed, same as with anything else. Because when we insert a new `Date`, all we get is a `uuid`:

```
INSERT Date {
  date := '22.44.10'
};
```

The output is just something like this: `{Object {id: 528941b8-f638-11ea-acc7-2fbb84b361f8}}` But what if we want to display its properties too? 

That's not hard: just wrap the whole entry in `SELECT ()`. Now that we've selected it, we can choose the properties to display, same as always:

```
SELECT ( # Start a selection
  INSERT Date { # Put the insert inside it
    date := '22.44.10'
} 
) # The bracket finishes the selection
{ # Now just choose the properties we want
  date,
  hour,
  awake
};
```

Now the output is more meaningful to us: `{Object {date: '22.44.10', hour: '22', awake: 'awake'}}` We know the date, the hour, and can see that the vampire is awake.

[Here is all our code so far up to Chapter 4.](chapter_4_code.md)

# Chapter 5 - Jonathan tries to leave the castle

Here's what happens in this chapter:

>During the day, Jonathan decides to try to explore the castle but too many doors and windows are locked. He doesn't know how to get out, and wishes he could at least send Mina a letter. He pretends that there is no problem, and keeps talking to Dracula during the night. One night he sees Dracula climb out of his window and down the castle wall, like a snake, and now he is very afraid. A few days later he breaks one of the doors and finds another part of the castle. The room is very strange and he feels sleepy. He opens his eyes and sees three vampire women next to him. He can't move.

## std::datetime

Since Jonathan was thinking of Mina back in London, let's learn about `std::datetime` because it uses time zones. To create a datetime, you can just cast a string in ISO 8601 format with `<datetime>`. That format looks like this:

`'2020-12-06T22:12:10Z'`

YYYY-MM-DDTHH:MM:SSZ

The `T` inside there is just a separator, and the `Z` at the end means "zero timeline". That means that it is 0 different (offset) from UTC: in other words, it *is* UTC.

One other way to get a `datetime` is to use the `to_datetime()` function. [Here is its signature](https://edgedb.com/docs/edgeql/funcops/datetime/#function::std::to_datetime), which shows that there are multiple ways to make a `datetime` with this function. The easiest is probably the third, which looks like this: `std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime`

With this, our game could have a function that just generates integers for times that then get a proper time stamp from `to_datetime`. Let's imagine that it's 10:35 am in Castle Dracula and Jonathan is trying to escape on May 12 to send Mina a letter. In Romania the time zone is 'EEST' (Eastern European Summer Time). We'll use `to_datetime()` to generate this. We won't worry about the year, because the story takes place in the same year - we'll just use 2020 for convenience. We type this:

`SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');`

And get the following output:

`{<datetime>'2020-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

With this we can see the duration between events, because there is a `duration` type that comes from subtracting a datetime from another one. Let's see the exact number of seconds between one date in Central Europe and another in Korea:

```
SELECT to_datetime(2020, 5, 12, 6, 10, 0, 'CET') - to_datetime(2000, 5, 12, 6, 10, 0, 'KST');
```

This takes May 12 2020 6:10 am in Central European Time and subtracts May 12 2000 6:10 in Korean Standard Time. The result is: `{631180800s}`.

Now let's try something similar. Imagine Jonathan in Castle Dracula on May 12 at 10:35 am, trying to escape. On the same day, Mina is in London at 6:10 am, drinking her morning tea. They are in different time zones, so how many seconds passed in between the two events? We can use `WITH` to declare some variables to make it easier to read. In this sample, `jonathan_wants_to_escape` and `mina_has_tea` are both of type `std::datetime`, so we can subtract one from the other:

```
WITH 
  jonathan_wants_to_escape := to_datetime(2020, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := to_datetime(2020, 5, 12, 6, 10, 0, 'UTC'),
  SELECT jonathan_wants_to_escape - mina_has_tea;
```

The output is `{5100s}`. As long as we know the timezone, EdgeDB does the work for us.

## REQUIRED LINK

Now we need to make a type for the three female vampires. We will call the type `MinorVampire`. These have a link to the `Vampire` type, it needs to be `REQUIRED` because they only live while Dracula lives.

```
type MinorVampire extending Person {
  REQUIRED LINK master -> Vampire;
}
```

Now that it's required, we can't just give it a name. It will give us this error: `ERROR: MissingRequiredError: missing value for required link default::MinorVampire.master`.

```
INSERT MinorVampire {
  name := 'Woman 1',
  master := (SELECT Vampire Filter .name = 'Count Dracula'),
};
```

This works because there is only one 'Count Dracula' (remember, `REQUIRED LINK` is short for `REQUIRED SINGLE LINK`). If there were more than one, we would have to add `LIMIT 1`.

## DESCRIBE

Our `MinorVampire` type extends `Vampire`, and `Vampire` extends `Person`. Types can continue to extend other types, and it can be annoying to read each one to try to put them together in our mind. Here you can use `DESCRIBE` to show exactly what our type is made of. There are three ways to do it:

- `DESCRIBE TYPE MinorVampire` - the DDL description of a type. This only shows the declaration used to make it, so it won't show the information from `Vampire` or `Person`.
- `DESCRIBE TYPE MinorVampire AS SDL` - same thing, but in SDL (the language we have been using).
- `DESCRIBE TYPE MinorVampire AS TEXT` - this is what we want. It shows everything inside the type. When we enter that, it gives us everything:

```
{
  'type default::MinorVampire extending default::Vampire {
    required single link __type__ -> schema::Type {
        readonly := true;
    };
    optional single link lover -> default::Person;
    required single link master -> default::Vampire;
    optional multi link places_visited -> default::Place;
    optional single property age -> std::int16;
    required single property id -> std::uuid {
        readonly := true;
    };
    required single property name -> std::str;
};',
}
```

The parts that say `readonly := true` we don't need to worry about, as they are automatically generated. For everything else, we can see that we need a `name` and a `master`, and could add a `lover`, `age` and `places_visited` for these `MinorVampire`s.

[Here is all our code so far up to Chapter 5.](chapter_5_code.md)

# Chapter 6 - Still no escape

>Jonathan can't move and the women vampires are next to him. Dracula runs into the room and tells the women to leave: "You can have him later, but not tonight!" The women listen to him. Jonathan wakes up in his bed and it feels like a bad dream...but he sees that somebody folded his clothes, and he knows it was not just a dream. The castle has some visitors from Slovakia the next day, so Jonathan has an idea. He writes two letters, one to Mina and one to his boss. He gives the visitors some money and asks them to send the letters. But Dracula finds them, and burns them in front of Jonathan. Jonathan is still stuck in the castle.

There is not much new in this lesson when it comes to types, so let's look at improving our schema. Right now Jonathan Harker is still inserted like this:

```
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

This was fine when we only had cities, but now we have the `Place` and `Country` type. First we'll insert a `Country` type with `name := 'Romania'`. Then we'll make a new type called `OtherPlace` for places that aren't cities or countries. That's easy: `type OtherPlace extending Place;`

Then we'll insert an `OtherPlace` with `name := 'Castle Dracula'`. Now we don't just have cities. Finally we will insert a `Country` with `name := 'Slovakia'`, just in case.

We can insert Jonathan with `Place`, but then he'll get every `Place` in the database, including Slovakia. Let's make sure that Jonathan doesn't always get every `Place` in the database when we insert him. We can do a quick `SELECT` on all `Place` types matching the name:

```
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};
```

You'll notice that we just wrote the names in a set using `{}`, so we didn't need to use an array with `[]` to do it.

Now what if Jonathan ever escapes Castle Dracula and runs away to a new place? Let's pretend that he runs away to Slovakia. Of course, we can change his `INSERT` signature to include `'Slovakia'`. But how about a quick update? For that we have the `UPDATE` and `SET` keywords. `UPDATE` starts the update, and `SET` is for the parts we want to change.

```
UPDATE NPC
  FILTER
  .name = 'Jonathan Harker'
  SET {
    places_visited += (SELECT Place FILTER .name = 'Slovakia')
};
```

And since Jonathan hasn't visited Slovakia, we can use `-=` instead of `+=` with the same `UPDATE` syntax to remove it now.

One other operator is `++`, which does concatenation (joining things together) instead of adding.

You can do simple operations like: ```SELECT 'My name is ' ++ 'Jonathan Harker';``` which gives `{'My name is Jonathan Harker'}`. Or you can do more complicated concatenations as long as you continue to join strings to strings:

```
SELECT 'A character from the book: ' ++ (SELECT NPC.name) ++ ', who is not ' ++ (SELECT Vampire.name);
```

This prints:

```
{
  'A character from the book: Jonathan Harker, who is not Count Dracula',
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
}
```

(The concatenation operator works on arrays too, putting them into a single array. So `SELECT ['I', 'am'] ++ ['Jonathan', 'Harker'];` gives `{['I', 'am', 'Jonathan', 'Harker']}`.)

Let's also change the `Vampire` type to link it to `MinorVampire` from that side instead. You'll remember that Count Dracula is the only real vampire, while the others are of type `MinorVampire`. That means we need a `MULTI LINK`:

```
type Vampire extending Person {            
  property age -> int16;
  MULTI LINK slaves -> MinorVampire;
}
```

Then we can `INSERT` the `MinorVampire` type at the same time as we insert the information for Count Dracula. But first let's remove the link from `MinorVampire`, because we don't want two objects linking to each other. There are two reasons for that:

- When we declare a `Vampire` it has `slaves`, but if there are no `MinorVampire`s yet then it will be empty: {}. And if we declare the `MinorVampire` type first it has a `master`, but if we declare them first then their `master` (a `REQUIRED LINK`) will not be there.
- If both types link to each other, we won't be able to delete them if we need to. The error looks something like this:

```
ERROR: ConstraintViolationError: deletion of default::Vampire (cc5ee436-fa23-11ea-85e0-e78b548f5a59) is prohibited by link target policy

DETAILS: Object is still referenced in link master of default::MinorVampire (cc87c78e-fa23-11ea-85e0-8f5149329e3a).
```

So first we simply change `MinorVampire` to a type extending `Person`:

```
type MinorVampire extending Person {
}
```

and then we create them all together with Count Dracula like this:

```
INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (INSERT MinorVampire {
      name := 'Woman 1',
  }),
    (INSERT MinorVampire {
     name := 'Woman 2',
  }),
    (INSERT MinorVampire {
     name := 'Woman 3',
  }),
 }
};
```

Then when we `select Vampire` like this:

```
SELECT Vampire {
  name,
  slaves: {name}
};
```

We have a nice output that shows them all together:

```
Object {
  name: 'Count Dracula',
  slaves: {Object {name: 'Woman 1'}, Object {name: 'Woman 2'}, Object {name: 'Woman 3'}},
  },
```

What do we do if we want the same output in json? That's easy: just cast using `<json>`. Any type in EdgeDB (except `bytes`) can be cast to json this easily:

```
SELECT <json>Vampire {
  name,
  slaves: {name}
};
```

The output is:

```
{
  "{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Woman 1\"}, {\"name\": \"Woman 2\"}, {\"name\": \"Woman 3\"}]}",
}
```

[Here is all our code so far up to Chapter 6.](chapter_6_code.md)

# Chapter 7 - Jonathan finally "leaves" the castle

> Jonathan sneaks into Dracula's room during the day and sees him sleeping inside a coffin. Now he knows that he is a vampire. Count Dracula says that he will leave tomorrow, and Jonathan asks to leave now. Dracula says, "Fine, if you wish..." and opens the door: but there are a lot of wolves outside. Jonathan knows that Dracula called the wolves, and asks him to close the door. Jonathan hears Dracula tell the women that they can have him tomorrow after he leaves. The next day Dracula's friends take him away (he is inside a coffin), and Jonathan is alone. Soon it will be night, and the doors are locked. He decides to escape out the window, because it is better to die by falling than to be alone with the vampire women. He writes "Good-bye, all! Mina!" and begins to climb the wall.

While Jonathan climbs the wall, we can continue to work on our database schema. In our book, no character has the same name. This is a good time to put a [constraint](https://edgedb.com/docs/datamodel/constraints#ref-datamodel-constraints) on `name` in the `Person` type. A `constraint` is a limitation, which we saw already in `age` for humans that can only go up to 120. For `name` we can give it another one called `constraint exclusive`. With that, no two objects of the same type can have the same name. You can put a `constraint` in a block after the type, like this:

```
abstract type Person {
  required property name -> str { ## Add a block
      constraint exclusive;       ## and the constraint
  }
  MULTI LINK places_visited -> Place;
  LINK lover -> Person;
}
```

Now we know that there will only be one `Jonathan Harker`, `Mina Murray`, and so on. In real life this is often useful for email addresses, User IDs, and so on. In our database we'll also add `constraint exclusive` to `Place` because those are also all unique.

Let's also think about our game mechanics a bit. The book says that Dracula is extremely strong, but Jonathan can't open the doors. We can think of doors as having a strength, and people having strength as well. If the person has greater strength than the door, then he or she can open it. So we'll create a type `Castle` and give it some doors. For now we only want to give it some "strength" numbers, so we'll just make it an `array<int16>`:

```
type Castle extending Place {
    property doors -> array<int16>;
}
```

Then we'll say there are three main doors to enter and leave Castle Dracula, so we `INSERT` it as follows:

```
INSERT Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};
```

Then we will also add a property `strength -> int16;` to our `Person` type. It won't be required because we don't know the strength of everybody in the book...though maybe later on we will if we want to make every character in the book into a character for the game.

When we alter the type we can just change it and do a migration again, as always. But if we don't want to do a migration, we can use some DDL to do it. You'll remember that DDL is not very good for migrations and schema, but is great for small changes. Here is how we add `strength` to `Person`:

```
ALTER TYPE Person {
  CREATE PROPERTY strength -> int16
};
```

Now we'll give Jonathan a strength of 5. We'll use `UPDATE` and `SET` like before:

```
UPDATE Person FILTER .name = 'Jonathan Harker'
  SET {
    strength := 5
};
```

Great. Now we can see if Jonathan can break out of the castle. To do that, he needs to have a strength greater than that of a door. Of course, we know that he can't do it but we want to make a query that can give the answer.

There is a function called `min()` that gives the minimum value of a set, so we can use that. If his strength is higher than the door with the smallest number, then he can escape. This almost works, but not quite:

```
WITH 
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  weakest_door := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
    SELECT jonathan_strength > min(weakest_door);
```

Here's the error:

```
error: operator '>' cannot be applied to operands of type 'std::int16' and 'array<std::int16>'
```

We can [look at the function signature](https://edgedb.com/docs/edgeql/funcops/set#function::std::min) to see the problem:

```
std::min(values: SET OF anytype) -> OPTIONAL anytype
```

So it needs a set, so something in curly brackets. We can't just put curly brackets around the array, because then it becomes a set of one item (one array). So `SELECT min({[5, 6]});` just returns `{[5, 6]}` because that is the minimum value of the one array.

That also means that `SELECT min({[5, 6], [2, 4]});` will work by giving us `{[2, 4]}`. But that's not what we want.

Instead, what we want to use is [array_unpack()](https://edgedb.com/docs/edgeql/funcops/array#function::std::array_unpack) which takes an array and unpacks it into a set. So we'll use that on `weakest_door`:

```
WITH 
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
    SELECT jonathan_strength > min(array_unpack(doors));
```

That gives us `{false}`. Perfect! Now we know that Jonathan will have to climb out the window to escape.

Along with `min()` there is of course `max()`. `len()` and `count()` are also useful: `len()` gives you the length of an object, and `count()` the number of them. Here are two examples with these functions:

```
SELECT (NPC.name, 'Name length is: ' ++ <str>len(NPC.name));
```

Don't forget that we need cast with `<str>` because `len()` returns an integer, and EdgeDB won't concatenate a string to an integer. This prints:

```
{
  ('The innkeeper', 'Name length is: 13'),
  ('Mina Murray', 'Name length is: 11'),
  ('Jonathan Harker', 'Name length is: 15'),
}
```

The other example is with `count()`, and also has a cast to a `<str>`:

```
SELECT 'There are ' ++ <str>(SELECT count(Place) - count(Castle)) ++ ' more places than castles';
```

It prints: `{'There are 6 more places than castles'}`.

In a few chapters we will learn how to use `CREATE FUNCTION` to make queries shorter.

[Here is all our code so far up to Chapter 7.](chapter_7_code.md)

# Chapter 8 - Dracula takes the boat to England

We are finally away from Castle Dracula. Here is what happens in this chapter:

> A boat leaves from the city of Varna. It has a **captain, first mate, second mate, cook**, and **five crew**. Inside is Dracula, but they don't know that he's there. Every night Dracula leaves his coffin, and every night one of the men disappears. They are afraid but don't know what to do. One of them sees Dracula but the others don't believe him. On the last day the ship gets close to England and the captain is alone. He ties his hands to the wheel so that the ship will go straight even if Dracula finds him. The next day the ship hits the beach, all the men are dead and Dracula turns into a wolf and runs onto the shore. People find the captain's notebook in his hand and start to read the story.

Let's learn about multiple inheritance. We know that you can `extend` a type on another, and we have done this many times: `Person` on `NPC`, `Place` on `City`, etc. Multiple inheritance is doing this with more than one type. We'll try this with the ship's crew. The book doesn't give them any names, so we will give them numbers instead. Most `Person` types won't need a number, so we'll create this:

```
abstract type HasNumber {
    required property number -> int16;
}
```

We will also remove `required` from `name` for the `Person` type. Not every `Person` type will have a name now, and we trust ourselves enough to input a name if there is one. We will of course keep it `exclusive`.

Now we can use multiple inheritance for the `Crewman` type. It looks like this:

```
type Crewman extending HasNumber, Person {
}
```

Now that we have this type and don't need a name, it's super easy to insert our crewmen thanks to `count()`. We just do this each time:

```
INSERT Crewman {
  number := count(DETACHED Crewman) + 1
};
```

So if there are no `Crewman` types, he will get the number 1. The next will get 2, and so on. So we do this five times and `SELECT Crewman{number};`. It gives us:

```
{
  Object {number: 1},
  Object {number: 2},
  Object {number: 3},
  Object {number: 4},
  Object {number: 5},
}
```

Next is the `Sailor` type. The sailors have ranks, so first we will make an enum for that:

```
scalar type Rank extending enum<'Captain', 'First mate', 'Second mate', 'Cook'>;
```

And then a `Sailor` type that uses `Person` and this enum:

```
type Sailor extending Person {
    property rank -> Rank;
}
```

Then finally a `Ship` type to hold them all.

```
type Ship {
  name -> str;
  MULTI LINK sailors -> Sailor;
  MULTI LINK crew -> Crewman;
}
```

Now to insert the sailors we just give them each a name and choose a rank from the enum:

```
INSERT Sailor {
  name := 'The Captain',
  rank := 'Captain'
};
```

We do that for the Captain, First Mate, Second Mate, and Cook.

```
INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

Then we can look up the `Ship` to make sure that the whole crew is there:

```
SELECT Ship {
  name,
  sailors: {
    name,
    rank,
    },
  crew: {
    number
    },
};
```

The result is:

```
{
  Object {
    name: 'The Demeter',
    sailors: {
      Object {name: 'Petrofsky', rank: First mate},
      Object {name: 'The Second Mate', rank: Second mate},
      Object {name: 'The Cook', rank: Cook},
      Object {name: 'The Captain', rank: Captain},
    },
    crew: {
      Object {number: 1},
      Object {number: 2},
      Object {number: 3},
      Object {number: 4},
      Object {number: 5},
    },
  },
}
```

So now we have quite a few types that extend the `Person` type, many with their own properties. The `Crewman` type has a property `number`, while the `NPC` type has a property called `age`. But because `Person` doesn't have all of these, this query won't work:

```
SELECT Person {
  name,
  age,
  number,
};
```

The error is `ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`. Luckily there is an easy fix for this using `IS` inside square brackets:

```
SELECT Person {
  name,
  [IS NPC].age,
  [IS Crewman].number,
};
```

Now it will only look for `age` for the `NPC` type and `number` for the `Crewman` type. The output is now quite large, but here is part of it:

```
{
  Object {name: 'Woman 1', age: {}, number: {}},
  Object {name: 'The innkeeper', age: 30, number: {}},
  Object {name: 'Mina Murray', age: {}, number: {}},
  Object {name: {}, age: {}, number: 1},
  Object {name: {}, age: {}, number: 2},
}
```

This is pretty good, but the output doesn't show us the type for each of them. To refer to self in a query in EdgeDB you can use `__type__`. Calling just `__type__` will just give a `uuid` though, so we need to add `{name}` to indicate that we want the name of the type. All types have this `name` field that you can access if you want to show the object type in a query.

```
SELECT Person {
  __type__: { 
    name      # Name of the type inside module default
  },
  name, # Person.name
  [IS NPC].age,
  [IS Crewman].number,
};
```

Choosing the five objects from before from the output, it now looks like this:

```
{
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Woman 1', age: {}, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'The innkeeper', age: 30, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'Mina Murray', age: {}, number: {}},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 1},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 2},
}
```

Once again we also see that there is no `NULL`: even properties that aren't part of a type return `{}`.

[Here is all our code so far up to Chapter 8.](chapter_8_code.md)

# Chapter 9 - Strange events in England

Introduction part 1:

> We still don't know where Jonathan is, and the ship is on its way to England. Meanwhile, Mina Harker is writing letters to her friend Lucy Westenra. Lucy has three boyfriends (named Dr. John Seward, Quincey Morris, and Arthur Holmwood) who want to marry her....

It looks like we have some more people to insert. But first, let's think about the ship a little more. Everyone on the ship was killed by Dracula, but we don't want to delete the crew because will be a part of our game. The book tells us that the ship left on the 6th of July, and the last person (the captain) died on the 4th of August (in 1887). This is a good time to add a `first_appearance` and `last_appearance` property to the `Person` type. We will choose `last_appearance` instead of `death`, because for the game it doesn't matter: we just want to have the right characters in the game at the right times.

For these two properties we will use `cal::local_date` because we can use the year 1887 for it. There is also `cal::local_datetime` that includes time, but we should be fine with just the date. (There is also a `cal::local_time` type with just the time of day)

We won't insert all the `first_appearance` and `last_appearance` values here, but the format looks like this:

```
INSERT Crewman {
  number := count(DETACHED Crewman) +1,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
};
```

And the easiest way to set them all is to use the `UPDATE` and `SET` syntax if we know the same date for all of them.

```
UPDATE Crewman
  SET {
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16)
};      
```

This will of course depend on our game. Can a `PC` actually visit the ship during the trip to England? If not, then we can just choose a rough date as above.

The [cal::to_local_date](https://edgedb.com/docs/edgeql/funcops/datetime#function::cal::to_local_date) function is the easiest way, as it allows you to simply insert three numbers to get a date. You can also cast from a string using `<cal::local_date>`, but the string must be in ISO8601 format.

Now let's get back to inserting the new characters. First we'll insert Lucy:

```
INSERT NPC {
  name := 'Lucy Westenra',
  places_visited := (SELECT City FILTER .name = 'London')
};
```

Hmm, it looks like we're doing a lot of work to insert 'London' every time. We have three characters left and they will all be from London too. Let's make London the default for `places_visited` for `NPC`. To do this we will need two things: `default` to declare a default, and the keyword `OVERLOADED`. `OVERLOADED` is because we are changing what we inherited from `Person`, and need to specify that.

```
type NPC extending Person {
  property age -> HumanAge;
  OVERLOADED MULTI LINK places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

So now we can enter all three of our new characters at the same time. To do this we can use a `FOR` loop, followed by the keyword `UNION`. After `FOR` we choose the variable name to use in the later part of the query.

```
FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  UNION (
    INSERT NPC {
    name := character_name,
    lover := (SELECT Person FILTER .name = 'Lucy Westenra'),
});
```

Then we can check to make sure that it worked:

```
SELECT NPC {
  name,
  places_visited: {
    name,
  },
  lover: {
  name,
  },
} FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'};
```

And we see them all connected to Lucy now:

```
  Object {
    name: 'John Seward',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
  Object {
    name: 'Quincey Morris',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
  Object {
    name: 'Arthur Holmwood',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
}
```

Now it's time to update Lucy with three lovers. But `LINK lover` in the `Person` type isn't set as MULTI so we'll have to change that. Let's try it with DDL again. You can see the usage [in the documentation here](https://edgedb.com/docs/edgeql/ddl/links#alter-link). We'll follow that:

```
ALTER TYPE Person {
  ALTER LINK lover {
    SET MULTI
  }
};
```

The output is `OK: ALTER`.

Great, now we can update Lucy:

```
UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (
    SELECT Person FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};
```


Now we'll select her to make sure it worked. We'll use `LIKE` this time:

```
SELECT NPC {
  name,
  lover: {
  name
  }
} FILTER .name LIKE 'Lucy%';
```

And this does indeed print her out with her three lovers.

```
{
  Object {
    name: 'Lucy Westenra',
    lover: {
      Object {name: 'John Seward'},
      Object {name: 'Quincey Morris'},
      Object {name: 'Arthur Holmwood'},
    },
  },
}
```

Okay, let's read the rest of the introduction about Lucy:

>...She chooses to marry Arthur Holmwood, and says sorry to the other two. Fortunately, the three men become friends with each other. Dr. Seward is sad and tries to concentrate on his work. He is a psychiatrist who is studying a Renfield, a man who believes that he can get power from living things by eating them. He's not a vampire, but seems to act similar sometimes.

Oops! Looks like she doesn't have three lovers anymore. Now we'll have to update her to only have Arthur:

```
UPDATE NPC FILTER .name = 'Lucy Westenra'
  SET {
    lover := (SELECT NPC FILTER .name = 'Arthur Holmwood'),
};
```

And then remove her from the other two:

```
UPDATE NPC FILTER .name in {'John Seward', 'Quincey Morris'}
  SET {
    lover := {} # 😢
};
```

Looks like we are mostly up to date now. The only thing left is to insert the mysterious Renfield. He is easy because he has no lover to `FILTER` for:

```
INSERT NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1887, 5, 26),
};
```

But he has some sort of relationship to Dracula, similar to the `MinorVampire` type but different. We will have to think about that later.

# Chapter 10 - Terrible events in Whitby

> Mina Murray travels from London to Whitby to see Lucy. One night there is a huge storm and a ship arrives - it's the Demeter, carrying Dracula. Lucy later begins to sleepwalk at night and looks very pale, and always says strange things. One night she watches the sun go down and says: "His red eyes again! They are just the same." Mina is worried and asks Dr. Seward for help. He doesn't know what the problem is and calls his old teacher Abraham Van Helsing, who arrives from the Netherlands to help her. Van Helsing examines Lucy. Then he turns to the others and says, "Listen. I have something to tell you that you might not believe..."

We finally have a new city: Whitby, up in the northeast. Right now our `City` type just extends `Place`. This could be a good time to give it a `property population` which can help us estimate the city size in our game. It will be an `int64` to give us the size we need.

Here's the approximate population for our three cities at the time of the book:

- Buda-Pesth: 402706
- London: 3500000
- Munich: 230023
- Whitby: 14400
- Bistritz: 9100

Inserting Whitby is easy enough:

```
INSERT City {
  name := 'Whitby',
  population := 14400
};
```

But for the rest of them it would be nice to update everything at the same time. If we have all the data together we can do it with a `FOR` loop again. The data in this case is a set of `tuple<str, int64>`, so this format:

`('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)`

You'll remember that to access part of an array, string etc. we use square brackets. So `SELECT ['Mina Murray', 'Lucy Westenra'][1];` will give the output `{'Lucy Westenra'}` (index number 1).

You can also slice strings with the same square brackets by using a colon to indicate the starting and ending index. For example:

```
SELECT NPC.name[0:10];
```

This prints the first ten letters of every NPC's name:

```
{
  'Jonathan H',
  'The innkee',
  'Mina Murra',
  'John Sewar',
  'Quincey Mo',
  'Lucy Weste',
  'Arthur Hol',
}
```

And the same can be done with a negative number to indicate how many characters from the end you want to slice. For example:

```
SELECT NPC.name[2:-2];
```

This prints from index 2 up to 2 indexes away from the end:

```
{
  'nathan Hark',
  'e innkeep',
  'na Murr',
  'hn Sewa',
  'incey Morr',
  'cy Westen',
  'thur Holmwo',
}
```

So that's how indexing and slicing works for every type except tuples, which use `()`. Tuples are different because they are more like individual object types with fields. This lets tuples hold different types together: `string`s with `array`s, `int64`s with `float32`s, anything. So this is completely fine:

```
SELECT {('Bistritz', 9100, cal::to_local_date(1887, 5, 6)), ('Munich', 230023, cal::to_local_date(1887, 5, 8))};
```

The output is:

```
{
  ('Bistritz', 9100, <cal::local_date>'1887-05-06'),
  ('Munich', 230023, <cal::local_date>'1887-05-08'),
}
```

But now that the type is set (this one is of type `tuple<str, int64, cal::local_date>`) you can't mix it up with other tuple types. So this is not allowed:

```
SELECT {(1, 2, 3), (4, 5, '6')};
```

EdgeDB will give an error because it won't try to work with tuples that are of different types. It complains:

```
ERROR: QueryError: operator 'UNION' cannot be applied to operands of type 'tuple<std::int64, std::int64, std::int64>' and 'tuple<std::int64, std::int64, std::str>'
  Hint: Consider using an explicit type cast or a conversion function.
```

In the above example we could easily just cast the last string into an integer and EdgeDB will be happy again: `SELECT {(1, 2, 3), (4, 5, <int64>'6')};`.

To access the fields of a tuple you still start from the number 0, but after a `.` instead of inside `[]`. Now that we know all this, we can update all our cities at the same time. It looks like this:

```
for data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
  UNION (
    UPDATE City FILTER .name = data.0
    SET {
    population := data.1
});
```

So it sends each tuple into the FOR loop, filters by the string (which is `data.0`) and then updates with the population (which is `data.1`).

Now that we have some numbers, we can do some math on them. Here we order them by population:

```
SELECT City {
  name,
  population
} ORDER BY .population DESC;
```

This returns:

```
{
  Object {name: 'London', population: 3500000},
  Object {name: 'Buda-Pesth', population: 402706},
  Object {name: 'Munich', population: 230023},
  Object {name: 'Whitby', population: 14400},
  Object {name: 'Bistritz', population: 9100},
}
```

What's DESC? It means descending, so largest first. If you don't write DESC then it will assume that you meant ascending. If you want to specify that, you can also write ASC.

For some actual math, you can check out the functions in `std` [here](https://edgedb.com/docs/edgeql/funcops/set#function::std::sum) as well as `math` [here](https://edgedb.com/docs/edgeql/funcops/math#function::math::stddev). Let's do a single big query to show some of them all together. To make the output nice, we will write it together with strings explaining the results and then cast them all to `<str>` so we can join them together using `++`.

```
WITH cities := City.population
  SELECT (
   'Number of cities: ' ++ <str>count(cities),
   'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
   'Total population: ' ++ <str>sum(cities),
   'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
   'Average population: ' ++ <str>math::mean(cities),
   'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
   'Standard deviation: ' ++ <str>math::stddev(cities)
);
```

The result is:

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

In the letter from Dr. Van Helsing to Dr. Seward, he starts it as follows:

`Letter, Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc., to Dr. Seward.`

This might be a good time to change our `Person` type. We have people with a lot of different types of names, in this order:

Title | First name | Last name | Degree

So there is 'Count Dracula' (title and name), 'Dr. Seward' (title and name), 'Dr. Abraham Van Helsing, M.D, Ph. D. Lit.' (title + first name + last name + degrees), and so on. But we also have characters that don't quite have a first and last name ('Woman 1', 'The Innkeeper') so it might not be a good idea to change the `name` property. But in our game we might have characters writing letters or talking to each other, and they will have to use things like titles and degrees.

We will add these properties to `Person`:

```
property title -> str;
property degrees -> str;
property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;
```

We could try to do something fancier with `degrees` by making it an `array<str>` for each degree, but our game probably doesn't need that much precision. We are just using this for our conversation engine.

Now it's time to insert Van Helsing:

```
INSERT NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```

Now we can make use of these properties to liven up our conversation engine in the game. For example:

```
WITH helsing := (SELECT NPC filter .name ILIKE '%helsing%')
  SELECT(
 'There goes ' ++ helsing.name ++ '.',
 'I say! Are you ' ++ helsing.conversational_name ++ '?',
 'Letter from ' ++ helsing.pen_name ++ ',\n\tI am sorry to say that I bring bad news about Lucy.');
```

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

# Chapter 11

