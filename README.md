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

Inside this we add the properties for our `Person` type. Use `required property` if the type needs it, and just `property` if it is optional. 

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

Hmm. We can see already that our schema needs some improvement:

- We have a `Person` type, and a `City` type,
- The `Person` type has the property `places_visited` which contains the names of the cities, but is not linked to any `City` type. There should be a relationship between these two.

We'll change that in a moment by changing `array<str>` from a property to something called `multi link` to the `City` type which will actually join them together. But first let's look a bit closer at what happens when we use `INSERT`.

As you can see, `str`s are fine with unicode letters like ț. Even emojis are just fine: you could create a `City` called '🤠' if you wanted to.

EdgeDB also has a byte literal type that lets you get the bytes of a string, but in this case it must be a character that is 1 byte long. You create it by adding a `b` in front of the string:

```
SELECT b'Bistritz';
{b'Bistritz'}
```

But because the characters must be 1 byte long, only ASCII works for this type. So the name in `modern_name` will generate an error:

```
SELECT b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
```

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

So now the last thing left to do is change the properties for `Person`. Right now, `places_visited` gives us the names we want, but it makes more sense to link `Person` and `City` together. We'll change `Person` to this:

```
type Person {
  required property name -> str;
  multi link places_visited -> City;
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

Right away we see that we could add another property to `City`, and will write this: `property important_places -> array<str>;` It will now look like this:

```
type City {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
```

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
  multi link places_visited -> City;
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
  multi link places_visited -> City;
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

We are now ready to make Dracula. Now, `places_visited` is still defined as a `Place`, and that includes many things: London, Bistritz, Hungary, etc. We only know that Dracula has been in Romania, so we can do a quick `FILTER` instead, inside `()`:

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
  multi link places_visited -> City;
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

But we don't want to have to read every line ourselves - we just want `true` or `false`. Now we'll add the computable to the query, using `EXISTS`. Because there is no null in EdgeDB, `EXISTS` gives `{false}` if it returns an empty set and `{true}` otherwise:

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
  multi link places_visited -> City;
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

With this, our game could have a function that generates integers for times that then use `to_datetime` to get get a proper time stamp. Let's imagine that it's 10:35 am in Castle Dracula and Jonathan is trying to escape on May 12 to send Mina a letter. In Romania the time zone is 'EEST' (Eastern European Summer Time). We'll use `to_datetime()` to generate this. We won't worry about the year, because the story takes place in the same year - we'll just use 2020 for convenience. We type this:

`SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');`

And get the following output:

`{<datetime>'2020-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

With this we can see the duration between events, because there is a `duration` type that you can get by subtracting a datetime from another one. Let's see the exact number of seconds between one date in Central Europe and another in Korea:

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

- `DESCRIBE TYPE MinorVampire` - the [DDL (data definition language)](https://www.edgedb.com/docs/edgeql/ddl/index/) description of a type. DDL is a lower level language than SDL, the language we have been using. It is more explicit and less convenient, but can be useful for quick changes. We won't look at DDL in this course but later on you might find it useful sometimes later on. For example, with it you can quickly create functions without needing to do a migration. And if you understand SDL it will not be hard to pick up some tricks in DDL.
- `DESCRIBE TYPE MinorVampire AS SDL` - same thing, but in SDL.
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

And for a *really* long output, try typing `DESCRIBE MODULE default` (with `AS SDL` or `AS TEXT` if you want). You'll get an output showing the whole module we've built so far.

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

Let's also change the `Vampire` type to link it to `MinorVampire` from that side instead. You'll remember that Count Dracula is the only real vampire, while the others are of type `MinorVampire`. That means we need a `multi link`:

```
type Vampire extending Person {            
  property age -> int16;
  multi link slaves -> MinorVampire;
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
  multi link places_visited -> Place;
  link lover -> Person;
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
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
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

Hmm, it looks like we're doing a lot of work to insert 'London' every time. We have three characters left and they will all be from London too. Let's make London the default for `places_visited` for `NPC`. To do this we will need two things: `default` to declare a default, and the keyword `overloaded`. `overloaded` is because we are changing what we inherited from `Person`, and need to specify that.

```
type NPC extending Person {
  property age -> HumanAge;
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

So now we can enter all three of our new characters at the same time. To do this we can use a `FOR` loop, followed by the keyword `UNION`. 

`UNION` is because it is the keyword used to join sets together. For example, this query:

```
WITH city_names := (SELECT City.name),
  castle_names := (SELECT Castle.name),
  SELECT city_names UNION castle_names;
```

joins the names together to give us the output `{'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Castle Dracula'}`.

Back to the `FOR` loop: after `FOR` we choose the variable name to use in the later part of the query.

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

By the way, now we can use this to insert our five `Crewman` types inside one `INSERT` instead of five. We'll change it to this now:

```
FOR n IN {1, 2, 3, 4, 5}
  UNION (
  INSERT Crewman {
  number := n
});
```

Now it's time to update Lucy with three lovers. But `LINK lover` in the `Person` type isn't set as MULTI so we'll have to change that. With this change made, we  can update Lucy:

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

Also, now that we know the keyword `overloaded`, we can remove our `HumanAge` type. Right now it looks like this:

```
scalar type HumanAge extending int16 {
      constraint max_value(120);
}
```

You will remember that we made this type because vampires can live forever, but humans only live up to 120. But now we can move `age` over to the `Person` type, and just use `overloaded` to add a constraint on it for the `NPC` type.

That lets us delete it from `Vampire` too:

```
type Vampire extending Person {    
  # property age -> int16; **Delete this one now**
  MULTI LINK slaves -> MinorVampire;
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
  strength := 10,
};
```

But he has some sort of relationship to Dracula, similar to the `MinorVampire` type but different. He is also quite strong, as we will see later. We will have to think about his relationship with Dracula later.

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

So that's how indexing and slicing works for every type except tuples, which use `()`. Tuples are different because they are more like individual object types with fields. This lets tuples hold different types together: `string`s with `array`s, `int64`s with `float32`s, anything.

So this is completely fine:

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

Now that we have some numbers, we can do some math on them. Here we order them by population with `ORDER BY`:

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

# Chapter 11 - What's wrong with Lucy?

> Dr. Van Helsing thinks that Lucy is being visited by a vampire. He doesn't tell the others yet because he knows they won't believe him, but tells them to close the windows and put garlic everywhere, and it works. But one day Lucy's mother thinks the room needs fresh air and opens the windows, and Lucy wakes up pale and sick again. Similar events happen many times, and every time the men give Lucy their blood to help her recover. Meanwhile, Renfield continues to try to eat living things and Dr. Seward can't understand him. And one day he simply did not talk to Dr. Seward, saying: “I don’t want to talk to you: you don’t count now; the Master is at hand.”

We are starting to see more and more events in the book with various characters. Some events have the three men and Dr. Van Helsing together, others have just Lucy and Dracula. Previous events had Jonathan Harker and Dracula, Jonathan Harker and the three women, and so on. Since we are making a game, we want to know which characters were in which place, and when. So let's try putting together an `Event` type.

This `Event` type is a bit long, but it needs to be the main type for our events in the game so it needs to be detailed. We can put it together like this:

```
scalar type EastWest extending enum<'E', 'W'>;

type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  property exact_location -> tuple<float64, float64>;
  property east_west -> EastWest;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ ' N ' ++ <str>.exact_location.1 ++ ' ' ++ <str>.east_west;
}
```    

You can see that most of the properties now are `required`, because an `Event` type is not useful to us for our game if it doesn't have the information we need. It will always need a description, a time, place, and people participating. The interesting part of this type is the `url` property which gives us an exact url for the location if we want. The events in the book take place in the north part of the planet, but sometimes they are east of Greenwich and sometimes west. We will always need to know if it is east or west so we create an enum with 'E' and 'W' and then use the computable `url` property to generate a link.

Let's insert one of the events in this chapter. It takes place on the night of September 11th when Dr. Van Helsing is trying to help Lucy.

```
INSERT Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1887, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1887, 9, 11, 23, 0, 0),
  place := (SELECT Place FILTER .name = 'Whitby'),
  people := (SELECT Person FILTER .name ILIKE {'%helsing%', '%westenra%', '%seward%'}),
  exact_location := (54.4858, 0.6206),
  east_west := 'W'
};
```

With all this information we can now find events by description, character, location, etc. The `description` property we can make as long as we want to make it easy to search, and we could even just paste in parts of the book if we wanted.

Now let's do a query for this event:

```
SELECT Event {
  description,
  start_time,
  end_time,
  place: {
  __type__: {
    name
    },
  name
   },
  people: {
  name
    },
  exact_location,
  url
} FILTER .description ILIKE '%garlic flowers%';
```

It generates a nice output that shows us everything about the event:

```
{
  Object {
    description: 'Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.',
    start_time: <cal::local_datetime>'1857-09-11T18:00:00',
    end_time: <cal::local_datetime>'1857-09-11T23:00:00',
    place: {Object {__type__: Object {name: 'default::City'}, name: 'Whitby'}},
    people: {
      Object {name: 'John Seward'},
      Object {name: 'Lucy Westenra'},
      Object {name: 'Abraham Van Helsing'},
    },
    exact_location: (54.4858, 0.6206),
    url: 'https://geohack.toolforge.org/geohack.php?params=54.4858 N 0.6206 W',
  },
}
```

The url works nicely too. Here it is: https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W You can see that it takes you directly to the city of Whitby.

We saw that Renfield is quite strong.

We could use this to experiment with making functions now. Because EdgeQL is strongly typed, you have to indicate the type going in and the type going out in the signature. A function that takes an int16 and gives a float64 for example would have this signature:

```
function does_something(input: int16) -> float64
```

The `->` skinny arrow is used to show the return value.

For the body of the function we do the following:

- Writing `using EdgeQL` (this is the only way to create functions at the moment)
- Write the function between `$$` to start and `$$` again to end
- Finish with a semicolon.

So let's write a function where we have two characters fight. We will make it very simple: the character with more strength wins.

```
function fight(one: Person, two: Person) -> str
  using EdgeQL $$
    SELECT one.name ++ ' wins!' IF one.strength > two.strength ELSE two.name ++ ' wins!'
$$;
```

So far only Jonathan and Renfield have the property `strength`, so let's put them up against each other:

```
WITH
  renfield := (SELECT Person filter .name = 'Renfield'),
  jonathan := (SELECT Person filter .name = 'Jonathan Harker')
    SELECT (
     fight(jonathan, renfield)
     );
```

It prints what we wanted to see: `{'Renfield wins!'}`

It might also be a good idea to add `LIMIT 1` when doing a filter for this function. Because EdgeDB returns sets, if it gets multiple results then it will use the function against each one. For example, say we forgot that there were three women in the castle and wrote this:

```
WITH
  the_woman := (SELECT Person filter .name ILIKE '%woman%'),
  jonathan := (SELECT Person filter .name = 'Jonathan Harker')
  SELECT (
    fight(the_woman, jonathan)
    );
```

It would give us this result:

```
{'Jonathan Harker wins!', 'Jonathan Harker wins!', 'Jonathan Harker wins!'}
```

In the book this is actually incorrect because Jonathan is weaker than each of the vampire women. But we haven't given them `strength` yet so even his `5` strength counts as more.

This is a good time to talk about Cartesian multiplication. When you multiply sets in EdgeDB you are given the Cartesian product, which looks like this:

![](Cartesian_product.png)

Source: [user quartl on Wikipedia](https://en.wikipedia.org/wiki/Cartesian_product#/media/File:Cartesian_Product_qtl1.svg)

This means that if we do a `SELECT` on `Person` for our `fight()` function, it will run the function as many times as the results for one multiplied by the other.

To demonstrate, let's put three objects in for each side of our function. We'll also make the output a little more clear:

```
WITH
  one := (SELECT Person FILTER .name in {'Jonathan Harker', 'Count Dracula', 'Arthur Holmwood'}),
  two := (SELECT Person FILTER .name in {'Renfield', 'Mina Murray', 'The innkeeper'}),
  SELECT(one.name ++ ' fights against ' ++ two.name ++ '. ' ++ fight(one, two));
```

Here is the output. Everyone fights once against everyone else:

```
{
  'Count Dracula fights against The innkeeper. The innkeeper wins!',
  'Count Dracula fights against Mina Murray. Mina Murray wins!',
  'Count Dracula fights against Renfield. Renfield wins!',
  'Jonathan Harker fights against The innkeeper. The innkeeper wins!',
  'Jonathan Harker fights against Mina Murray. Mina Murray wins!',
  'Jonathan Harker fights against Renfield. Renfield wins!',
  'Arthur Holmwood fights against The innkeeper. The innkeeper wins!',
  'Arthur Holmwood fights against Mina Murray. Mina Murray wins!',
  'Arthur Holmwood fights against Renfield. Renfield wins!',
}
```

And if you take out the filter and just write `SELECT Person` for the function, you will get well over 100 results.

# Chapter 12 - From bad to worse

There is no good news for our heroes this chapter:

> Van Helsing gives strict instructions on how to protect Lucy, but sometimes people don't follow all of them. When this happens Dracula manages to slip in as a cloud, drink her blood and sneak away before morning. Meanwhile, Renfield breaks out of his cell and attacks Dr. Seward with a knife. He cuts him with it, and the moment he sees the blood he stops attacking and tries to drink it. Dr. Seward's men take Renfield away and Dr. Seward is left confused and trying to understand him. He thinks there is a connection between him and the other events. That night, a wolf controlled by Dracula breaks the windows of Lucy's room and Dracula is able to get in again.

But there is good news for us, because we are going to keep learning about function overloading and Cartesian products.

First we should give a value for `.strength` for our characters. Jonathan Harker is actually quite strong (for a human), and has a strength of 5. We'll treat that as the maximum strength for a human. EdgeDB has a random function called `std::rand()` that gives a `float64` in between 0.0 and 1.0. There is another function called `round()` that rounds numbers, so we'll use that too, and finally cast it to an '<int16>'. Our input looks like this:
 
```
  SELECT <int16>round((random() * 5)); 
```

So now we'll use this to update our Person types and give them all a random strength.

```
WITH random_5 := (SELECT <int16>round((random() * 5)))
 # WITH isn't necessary - just making the query prettier
 
UPDATE Person
  FILTER NOT EXISTS .strength
  SET {
    strength := random_5 
};
```

And we'll make sure Count Dracula gets 20 strength, because he's Dracula:

```
UPDATE Vampire
FILTER .name = 'Count Dracula'
SET {
  strength := 20
  };
```

Now let's `SELECT Person.strength;` and see if it works:

```
{3, 3, 3, 2, 3, 2, 2, 2, 3, 3, 3, 3, 4, 1, 5, 10, 4, 4, 20, 4, 4, 4, 4}
```

Looks like it worked.

So now let's overload the `fight()` function so that more than one character can join together to fight another one. There are a lot of ways to do it, but we'll choose a simple one:

```
    function fight(names: str, one: int16, two: Person) -> str
  using EdgeQL $$
    SELECT names ++ ' win!' IF one > two.strength ELSE two.name ++ ' wins!'
$$;
```

So it's the same function name, except that we enter the names of the people (or team name) together, followed by their strength together, and then the `Person` they are fighting against.

Now Jonathan and Renfield are going to try to fight Dracula together. Good luck!

```
WITH
  jon_and_ren_strength := <int16>(SELECT sum(
    (SELECT NPC FILTER .name IN {'Jonathan Harker', 'Renfield'}).strength)
    ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

  SELECT fight('Jon and Ren', jon_and_ren_strength, dracula);
```

So did they...

```
{'Count Dracula wins!'}
```

No, they didn't win. How about four people?

```
WITH
  four_people_strength := <int16>(SELECT sum(
    (SELECT NPC FILTER .name IN {'Jonathan Harker', 'Renfield', 'Arthur Holmwood', 'The innkeeper'}).strength)
    ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

  SELECT fight('The four people', four_people_strength, dracula);
```

Much better:

```
{'The four people win!'}
```

So that's how function overloading works - you can create functions with the same name as long as the signature is different. Overloading is used a lot for existing functions, such as [sum](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::sum) which takes in all numeric types and returns the sum of the same type. [std::to_datetime](https://www.edgedb.com/docs/edgeql/funcops/datetime#function::std::to_datetime) has even more interesting overloading with all sorts of inputs to create a `datetime`.

`fight()` was pretty fun to make, but that sort of function is better done on the gaming side. Let's make a function that we might actually use. Here is a simple one that tells us if a `Person` type has visited a `Place` or not:

```
function visited(person: str, city: str) -> bool
      using EdgeQL $$
        WITH _person := (SELECT Person FILTER .name = person LIMIT 1),
        SELECT city IN _person.places_visited.name
      $$;
```

Now our queries are much faster:

```
edgedb> SELECT visited('Mina Murray', 'London');
{true}
edgedb> SELECT visited('Mina Murray', 'Bistritz');
{false}
```

And even more complicated queries are still quite readable:

```
SELECT(
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' ++ <str>visited('Jonathan Harker', 'Romania')
 );
```

This prints `{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`.

The documentation for creating functions [is here](https://www.edgedb.com/docs/edgeql/ddl/functions#create-function). You can see that you can create them with SDL or DDL but there is not much difference between the two.

Now let's learn more about Cartesian products in EdgeDB. Because the Cartesian product is used with sets, you might be surprised to see that when you put a `{}` empty set into an equation it will only return `{}`. For example, let's try to add the names of places that start with b and those that start with f.

```
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
  f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
  SELECT b_places.name ++ ' ' ++ f_places.name;
```

The output is:

```
{}
```

!! But a search for places that start with b gives us `{'Buda-Pesth', 'Bistritz'}`. Let's try manually concatenating just to make sure:

```
SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
```

The output is...

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast or a conversion function.
```

!!! This is an important point though: EdgeDB requires a cast for an empty set, because it won't try to guess at what type it is. Okay, one more time:

```
edgedb> SELECT {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
edgedb>  
```

Good, so we have manually confirmed that using `{}` with another set always returns `{}`. But what if we want to:

- Concatenate the two strings if they exist,
- Return what we have if one is an empty set?

To do that we can use the so-called coalescing operator, which is written `??`. Here is a quick example:

```
edgedb> SELECT {'Count Dracula is now in Whitby'} ?? <str>{};
{'Count Dracula is now in Whitby'}
edgedb>  
```

```
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
     SELECT 
       b_places.name ++ ' ' ++ f_places.name IF EXISTS b_places.name AND EXISTS f_places.name 
     ELSE 
       b_places.name ?? f_places.name;
```

This returns:

```
{'Buda-Pesth', 'Bistritz'}
```

That's better.

But remember, when we add or concatenate sets we are working with every item in each set separately. So if we change the query to search for places that start with b and m:

```
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     m_places := (SELECT Place FILTER Place.name ILIKE 'm%'),
     SELECT 
       b_places.name ++ ' ' ++ m_places.name IF EXISTS b_places.name AND EXISTS m_places.name 
     ELSE 
       b_places.name ?? m_places.name;
```

Then we'll get this:

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

instead of something like 'Buda-Peth, Bistritz, Munich'. To get that output, we have to do two more things:

- [array_agg](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_agg) to turn a set into an array, then
- [array_join](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_join) to turn the array into a string.

```
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     m_places := (SELECT Place FILTER Place.name ILIKE 'm%'),
     SELECT
     array_join(array_agg(b_places.name), ', ') ++ ', ' ++ array_join(array_agg(m_places.name), ', ') IF EXISTS b_places.name AND EXISTS m_places.name

      ELSE
        b_places.name ?? m_places.name;
```

Finally! The output is `{'Buda-Pesth, Bistritz, Munich'}`. Now with this more robust query we can use it on anything and don't need to worry about getting {} if we choose a letter like x. Let's look at every place that contains k or e:

```
WITH has_k := (SELECT Place FILTER Place.name ILIKE '%k%'),
     has_e := (SELECT Place FILTER Place.name ILIKE '%e%'),
     SELECT
     array_join(array_agg(has_k.name), ', ') ++ ', ' ++ array_join(array_agg(has_e.name), ', ') IF EXISTS has_k.name AND EXISTS has_e.name
     ELSE
    has_k.name ?? has_e.name;
```

This gives us the result:

```
{'Slovakia, Buda-Pesth, Castle Dracula'}
```

# Chapter 13 

> This time it was too late, and Lucy lies dying. Suddenly she opens her eyes and tells Arthur to kiss her. He tries, but Van Helsing grabs him and says "Don't you dare approach her!" It was not Lucy, but the vampire inside her that was talking. She dies, and Van Helsing puts a golden crucifix on her lips to stop her from moving (vampires can't move underneath one), but the nurse steals it when nobody is looking. Vampire Lucy starts walking around the town and biting children. Van Helsing tells the other people the truth, but Arthur can't believe him and becomes angry that he would insult Lucy by saying such things about her.

Looks like Lucy has become a `MinorVampire`. Right now `MinorVampire` is nothing special, just a type that extends `Person`:

```
type MinorVampire extending Person {
    }
```

Fortunately, she is effectively a new type as a `MinorVampire`, and the book says this too: it's not really Lucy anymore. So we can just give `MinorVampire` an optional link to `Person`:

```
type MinorVampire extending Person {
        link former_self -> Person;
}
```

One other thing we can remember to do is to give `last_appearance` for Lucy and `first_appearance` for Lucy as a `MinorVampire` the same date. First we will update Lucy with her `last_appearance`:

```
UPDATE Person filter .name = 'Lucy Westenra'
  SET {
  last_appearance := cal::to_local_date(1887,9,20)
};
```

Then we can add Lucy to the `INSERT` for Dracula. Note the first line where we create a variable called `lucy`. We then use that to bring in all the data for the `MinorVampire` based on her, a much better method than manually inserting all the information. It also includes her strength which is her human strength plus 5.

```
WITH lucy := (SELECT Person filter .name = 'Lucy Westenra' LIMIT 1)
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
    (INSERT MinorVampire {
     name := lucy.name,
     former_self := lucy,
     first_appearance := lucy.last_appearance,
     strength := lucy.strength + 5,
    }),
 }
};
```

With our `MinorVampire` types inserted that way, it's easy to find minor vampires that come from `Person` objects in the database:

```
SELECT MinorVampire {
  name,
  strength,
  first_appearance,
} FILTER .name IN Person.name AND .first_appearance IN Person.last_appearance;
```

This gives us:

```
{
  Object {
    name: 'Lucy Westenra',
    strength: 7,
    first_appearance: <cal::local_date>'1887-09-20',
  },
}
```

We could have just gone with `FILTER.name IN Person.name` but two filters is better if we have more and more characters later on. And if necessary we could switch to `cal::local_datetime` instead of `cal::local_date` to make sure that we have the exact time down to the minute. But for our book, we won't need to get that precise.

Without too much new to add, let's look at some tips for making queries.

- `DISTINCT`

Use this after `SELECT` to get only results that are not duplicates. Right now if we look at all the strength values for our `Person` objects with `SELECT Person.strength;` we get something like this:

```
{5, 4, 4, 4, 4, 4, 10, 2, 2, 2, 2, 2, 2, 2, 7, 5}
```

Change it to `SELECT DISTINCT Person.strength;` and the output will now be `{2, 4, 5, 7, 10}`.

- Getting `__type__` all the time

You will remember that we can use `__type__` to get the types of objects in a query, and that `__type__` always has `.name` that we can access to get the actual name (otherwise we will only get the uuid). We can use this to look at the type names inside `Person`:

```
{
  Object {name: 'default::Person'},
  Object {name: 'default::MinorVampire'},
  Object {name: 'default::Vampire'},
  Object {name: 'default::NPC'},
  Object {name: 'default::PC'},
  Object {name: 'default::Crewman'},
  Object {name: 'default::Sailor'},
}
```

Or we can use it in a regular query to return the types as well. Let's see what types there are that have the name `Lucy Westenra`:

```
Select Person {
  __type__: {
  name
  },
name
} filter .name = 'Lucy Westenra';
```

This shows us the objects that match, and of course they are `NPC` and `MinorVampire`.

```
{
  Object {__type__: Object {name: 'default::NPC'}, name: 'Lucy Westenra'},
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Lucy Westenra'},
}
```

But if you want to always get the type when you make a query, you can also just use `\set introspect-types on` to do it. Once you type that, you'll always see the type instead of `Object`. Now even a simple search like this will give us the type:

```
SELECT Person 
  {
name
} FILTER .name = 'Lucy Westenra';
```

Here's the output:

```
{default::NPC {name: 'Lucy Westenra'}, default::MinorVampire {name: 'Lucy Westenra'}}
```

- INTROSPECT

The word `introspect` we just used to get the type for every query is also its own keyword. Every type has `name`, `properties`, `links` and `target` that you can access. Let's give that a try and see what we get. We'll start with this on our `Ship` type, which is quite simple but has all four. The type is:

```
type Ship {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

The simplest `INTROSPECT` query is not useful but shows how it works: `SELECT (INTROSPECT Ship.name);` which gives us `{'default::Ship'}`. Note that `INTROSPECT` and the type go inside brackets.

Now let's put `name`, `properties` and `links` inside:

```
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links,
};
```

This gives us:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 4332bd76-0134-11eb-b20a-777e9cc80030},
      schema::Property {id: 43379841-0134-11eb-a427-dd28c7eae0ce},
    },
    links: {
      schema::Link {id: 4339bd5f-0134-11eb-bdca-21c31be0f932},
      schema::Link {id: 43360bf2-0134-11eb-b89c-a1211cb5d40e},
      schema::Link {id: 43383b6f-0134-11eb-86db-17f19f35ac2f},
    },
  },
}
```

Just like using `SELECT` on a type, if the output contains another type, property etc. we will just get an id. We will have to specify what we want there as well.

Eventually we end up using this sort of query to get the information we want:

```
SELECT (INTROSPECT Ship) {
   name,
   properties: {
     name,
     target: {
       name
       }
 },
 links: {
   name,
 target: {
   name
   },
 },
};
```

So what this will give is: 

1) The type name for `Ship`, 
2) The properties, and their names. But we also use `target`: a `target` is what a property actually is. For example, the target of `property name -> str` is `std::str`. And we want the target name too; without it we'll get an output like `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}`.
3) The links and their names, and the targets to the links...and the names of *their* targets too.

With all that together, we get something readable and useful. The output looks like this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: 'crew', target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: '__type__', target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'sailors', target: schema::ObjectType {name: 'default::Sailor'}},
    },
  },
}
```

This type of query seems complex (even in our simple schema it already goes four levels deep) but it really is just built up on top of adding extra details like `{name}` every time you get an output that only a machine can understand.

Plus, if the query isn't too complex (like ours), you might find it easier to read without so many new lines and indentation. Here's the same query written that way:

```
SELECT (INTROSPECT Ship) {
  name,
  properties: {name, target: {name}},
  links: {name, target: {name}},
};             
```

# Chapter 14 - Jonathan Harker returns

> Finally there is some good news. Jonathan Harker found his way to Budapest in August and then to a hospital, which sent Mina a letter. Jonathan recovered there and they took a train back to England, to the city of Exeter where they got married. Mina thinks that Lucy is still alive and sends her a letter that is never opened. Meanwhile, the men find vampire Lucy in a graveyard. Arthur finally believes Van Helsing and the rest also now believe that vampires are real. They manage to destroy vampire Lucy. Arthur is sad but happy to see that Lucy is no longer forced to be a vampire and can die in peace.

So we have a new city called Exeter, and adding it is of course easy: `INSERT City {name := 'Exeter', population := 40000};`. It doesn't have a `modern_name` that is different from the one in the book either.

Now that we know how to do introspection queries though, we can start to give our types `annotations`. An annotation is a string inside the type definition that gives us information about it. Let's imagine that in our game a `City` needs at least 50 buildings. By default, annotations can be called `title` or `description`. Let's use `description`:

```
type City extending Place {
  annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
  property population -> int64;
}
```

Now we can do an `INTROSPECT` query on it. We know how to do this from the last chapter - just add `: {name}` everywhere to get the inner details. Ready!

```
SELECT (INTROSPECT City) {
  name,
  properties: {name},
  annotations: {name}
};
```

Uh oh, not quite:

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {schema::Annotation {name: 'std::description'}},
  },
}
```

Ah, of course: the `annotations: {name}` part returns the name of the type, which is `std::description`. To get the value inside we write something else: `@value`. The `@` is used to directly access the value inside (the string) instead of just the type name. Let's try one more time:

```
SELECT (INTROSPECT City) {
 name,
 properties: {name},
 annotations: {
   name,
   @value
  }
};
```

Now we see the actual annotation:

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'Anything with 50 or more buildings is a city - anything else is an OtherPlace',
      },
    },
  },
}
```


What if we want an annotation with a different name besides `title` and `description`? That's easy, just use `abstract annotation` and give it a name. We want to add a warning so that's what we'll call it:

```
abstract annotation warning;
```

We'll imagine that it is important to know not to use `OtherPlace` for `Castle` types. Now `OtherPlace` has the following two annotations:

```
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
}
```

Now let's do an introspect query on just its name and annotations:

```
SELECT (INTROSPECT OtherPlace) {
  name,
  annotations: {name, @value}
};
```

And here it is:

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {name: 'std::description', @value: 'A place with under 50 buildings - hamlets, small villages, etc.'},
      schema::Annotation {name: 'default::warning', @value: 'Castles and castle towns do not count! Use the Castle type for that'},
    },
  },
}
```

A lot of characters are starting to die now, so let's think about that. We could come up with a method to see who is alive and who is dead, depending on a `cal::local_date`. First let's take a look at the `People` objects we have so far. We can easily count them with `count()`, which gives `{23}`. There is also a function called `enumerate()` that gives tuples of the index and the property that we choose. For example, if we use `SELECT enumerate(Person.name);` then we get this:

```
{
  (0, 'Renfield'),
  (1, 'The innkeeper'),
  (2, 'Mina Murray'),
# snip
  (14, 'Count Dracula'),
  (15, 'Woman 1'),
  (16, 'Woman 2'),
  (17, 'Woman 3'),
}
```

18! Oh, that's right: the `Crewman` objects don't have a name. We could of course try something fancy like this:

```
WITH 
  a := array_agg((SELECT enumerate(Person.name))),
  b:= array_agg((SELECT enumerate(Crewman.number))),
  SELECT (a, b);
```

(`array_agg()` is to avoid multiplying sets by sets)

But the result is less than satisfying:

```
{
  (
    [
      (0, 'Renfield'),
      (1, 'The innkeeper'),
      (2, 'Mina Murray'),
# snip
      (14, 'Count Dracula'),
      (15, 'Woman 1'),
      (16, 'Woman 2'),
      (17, 'Woman 3'),
    ],
    [(0, 1), (1, 2), (2, 3), (3, 4), (4, 5)],
  ),
}
```

Especially the `Crewman` types who are now just numbers. Better to give them names based on the numbers. This will be easy:

```
UPDATE Crewman
  SET {
    name := 'Crewman ' ++ <str>.number
};
```

So now we can see if they are dead or not. We'll compare a `cal::local_date` we input to `last_appearance` and if it's greater, then they are dead.

```
WITH p := (SELECT Person),
  date := <cal::local_date>'1887-08-16',
  SELECT(p.name, p.last_appearance, 'Dead on ' ++ <str>date ++ '? ' ++ <str>(date > p.last_appearance));  
```

Here is the output:

```
{
  ('Lucy Westenra', <cal::local_date>'1887-09-20', 'Dead on 1888-08-16? true'),
  ('Crewman 1', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 2', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 3', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 4', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 5', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
}
```

We could of course turn this into a function if we use it enough.

Finally, let's look at how to follow links in reverse direction. We know how to get Count Dracula's `slaves` by name with something like this:

```
SELECT Vampire {
 name,
 slaves: {name}
};
```

That shows us the following:

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Woman 1'},
      default::MinorVampire {name: 'Woman 2'},
      default::MinorVampire {name: 'Woman 3'},
      default::MinorVampire {name: 'Lucy Westenra'},
    },
  },
}
```

But what if we are doing the opposite and starting from `SELECT MinorVampire` and want to know about the `Vampire` type connected to it? Then we use `.<` instead of `.` and specify the type we are looking for: `[IS Vampire]`. All together, it looks like this:

```
SELECT MinorVampire.<slaves[IS Vampire] {
  name, 
  age
  };
```

Because it goes in reverse order, it is selecting the `Vampire` type with `.slaves` that are of type `MinorVampire`. Here is the output:

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```


# Chapter 15 - Time to start vampire hunting

> Jonathan meets with Van Helsing who tells him than his experience with Dracula was real and not a crazy dream. Jonathan becomes strong and confident again and they begin to search for Dracula. It turns out that the house called Carfax which Dracula bought is located right across from the hospital where Renfield lives and Dr. Seward works. They search the house when the sun is up and find boxes of earth in which Dracula sleeps. They destroy some of them so that Dracula can't use them anymore, but there are still many left. If they don't find and destroy the boxes, Dracula will be able to run away and rest at night and continue to terrorize London.

This chapter we learned something interesting about vampires: they need coffins (boxes for dead people) with holy earth to rest in during the day. Dracula brought 50 of them over by ship so that he could always have a place to hide and rest in London so he could terrorize the people. This is important for the mechanics of our game so we should create a type for this:

```
abstract type HasCoffins {
  required property coffins -> int16 {
    default := 0;
  }
}
```

Most places will not have a special vampire coffin, so the default is 0. `coffins` is just an `int16` because we want vampires to be close to a place if the number is 1 or greater. In the mechanics of our game we would probably give vampires an activity radius of about 100 km from a place with a coffin. That's because a horse-driven carriage can go about 25 kph, and a vampire can probably only go about 4 hours away from their coffins before they start to get nervous about getting home before the sun rises.

We will want to have a lot of types `extending` this. First with `Place` we can extend it for all types including `City`, `OtherPlace`, etc.:

```
abstract type Place extending HasCoffins {
  required property name -> str {
    constraint exclusive;
};
  property modern_name -> str;
  property important_places -> array<str>;
}
```

And we had Dracula's coffins on the ship so we'll extend for `Ship` as well.

```
type Ship extending HasCoffins {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

If we want, we can now make a quick function to test whether a vampire can enter a place:

```
function can_enter(person_name: str, place: HasCoffins) -> str
  using EdgeQL $$
    with vampire := (SELECT Person filter .name = person_name LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
        $$;   
```

Then we'll give London some coffins, 50 for now. Some get destroyed in this chapter but anything over 1 is fine for a vampire so it doesn't matter.

```
UPDATE City filter .name = 'London'
  SET {
    coffins := 50
 };
```

Then calling up our function:

```
SELECT can_enter('Count Dracula', (SELECT City filter .name = 'London'));
``` 

We get `{'Count Dracula can enter.'}`.

Some other options for improvement later on are:

- Move `name` from `Place` and `Ship` over to `HasCoffins`. Then the user could just enter a string, which the function would use to `SELECT` the type and then display its name, giving a result like "Count Dracula can enter London."
- Require a date in the function so that we can check if the vampire is dead or not first. For example, if we entered a date after Lucy died, the function would just remind us that `vampire.name` is dead already and won't be entering anywhere.
- Change the function `can_enter()` to take a vampire type instead of `Person`. Right now both `Vampire` and `MinorVampire` just extend `Person`, so our function needs to trust that the user is entering a string for a name of a vampire.

## Constraints

Let's look at two more constraints. We've seen `exclusive` and `max_value` already, but there are [some others](https://www.edgedb.com/docs/edgeql/sdl/constraints/#constraints) we can use as well. There is one called `max_len_value` that makes sure that a string doesn't go over a certain length. That will be good for our `PC` type, which we created but have only used once because we are still just using the Dracula book to populate a database. But `max_len_value()` will definitely be needed for `PC`. We'll change it to look like this:

```
type PC extending Person {
  required property transport -> Transport;
  overloaded required property name -> str {
    constraint max_len_value(30);
  }
}
```

Then when we try to insert a `PC` with a name that is too long, it will refuse with `ERROR: ConstraintViolationError: name must be no longer than 30 characters.`

One particularly flexible constraint is called `expression on`, which lets us add any expression we want (in brackets). Let's say we need a type `Lord` for some reason later on. We can constrain the type to make sure that its `name` always contains the string `Lord`. We can write it like this:

```
type Lord extending Person {
  constraint expression on (
    contains(__subject__.name, 'Lord') = true
    );
}
```

`__subject__` there refers to the type itself.

Now when we try to insert a `Lord` without it, it won't work:

```
INSERT Lord {
  name := 'Billy'
  # Other stuff..
};
```

Now let's try it with the word `Lord` inside. Let's do a `SELECT` and `INSERT` at the same time for this so we see the output of our `INSERT` right away. We'll change `Billy` to `Lord Billy` and say that Lord Billy has visited every place in our database.

```
SELECT (
 INSERT Lord {
 name := 'Lord Billy',
 places_visited := (SELECT Place),
 }) {
 name,
 places_visited: {
   name
   }
 };
 ```
 
Now that `.name` contains the substring `Lord`, it works like a charm:

```
{
  default::Lord {
    name: 'Lord Billy',
    places_visited: {
      default::Castle {name: 'Castle Dracula'},
      default::City {name: 'Whitby'},
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
      default::City {name: 'London'},
      default::Country {name: 'Romania'},
      default::Country {name: 'Slovakia'},
    },
  },
}
```

# Chapter 16 - Is Renfield telling the truth?

> Van Helsing meets with Renfield, and is surprised: Renfield one day turns out to be very educated and well-spoken, and knows all about Van Helsing's research. But the next day, Renfield doesn't want to talk to him and calls him an idiot. Arthur Holmwood's father has died, and now Arthur is the head of the house and is called Lord Godalming. He helps the team with his money to find the houses where Dracula has hidden his boxes. One night, Renfield tells Van Helsing and Dr. Seward to please trust him and let him leave. They won't let him, and he finally says: “Remember, later on, that I did what I could to convince you tonight.”

We're getting closer to the end of the book and there is a lot of data that we haven't entered yet. There is also a lot of data from the book that might be useful but we're not ready to organize yet. Instead, we could make a type that holds a date and strings from the book for us to search through later. Let's call it `BookExcerpt` (= part of a book).

```
  type BookExcerpt {
    required property date -> cal::local_datetime;
    required property excerpt -> string;
    index on (.excerpt);
    required link author -> Person
}
```

The `index on (.exerpt)` part is new, and means to create an index to make future queries faster. We could do this for certain other types too, and might for types like `Place` and `Person`. `index` is good in limited quantities, and you don't want to index everything: 

- It makes the queries faster, but increases the database size.
- This may make `insert`s and `update`s slower if you have too many.

This is maybe not surprising: if `index` were good to have for every property and every link on every type, EdgeDB would just do it automatically for everything.

Fortunately, the book Dracula is written from the point of view of everyone's letters so each one has a date and an author. The string in these entries are very long so we will only include the beginning and the end:

```
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 4, 0, 0),
  author := (SELECT Person FILTER .name = 'John Seward' LIMIT 1),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};
```

```
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 5, 0, 0),
  author := (SELECT Person FILTER .name = 'Jonathan Harker' LIMIT 1),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```

Then later on we could do something like the following if we want all the entries in order and turned into JSON.

```
SELECT <json>(SELECT BookExcerpt {
  date,
  author: {name},
  excerpt
} ORDER BY .date);
```

Output with exerpts snipped:

```
{
  "{\"date\": \"1887-10-01T04:00:00\", \"author\": {\"name\": \"John Seward\"}, \"excerpt\": \"Dr. Seward\'s Diary.\\n 1 October, 4 a.m... -- Just as we were about to leave the house...\\\"\"}",
  "{\"date\": \"1887-10-01T05:00:00\", \"author\": {\"name\": \"Jonathan Harker\"}, \"excerpt\": \"1 October, 5 a.m. -- I went with the party to the search...\"}",
}
```

After this, we can add a link to our `Event` type, which now looks like this:

```
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  multi link excerpt -> BookExcerpt; # Only this is new
  property exact_location -> tuple<float64, float64>;
  property east_west -> EastWest;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ ' N ' ++ <str>.exact_location.1 ++ ' ' ++ <str>.east_west;
}
```

Now `description` contains a short description that we write, while `excerpt` links to these longer pieces of text that come directly from the book.

When doing queries on our `BookExcerpt` type (or `BookExcerpt` via `Event`), the [functions for strings](https://www.edgedb.com/docs/edgeql/funcops/string) can be particularly useful. Here is one:

```
select BookExcerpt {
  excerpt,
  length := (<str>(SELECT len(.excerpt)) ++ ' characters'),
  the_date := (SELECT (<str>.date)[0:10]),
} FILTER contains(str_lower(.excerpt), 'mina');
```

It uses `len()` which is then cast to a string, and `str_lower()` to compare against `.excerpt()` by making it lowercase first. It also slices the `cal::local_datetime` into a string so it can just print indexes 0 to 10. Here is the output:

```
{
  Object {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1887-10-01',
  },
}
```

Some other functions for strings are:

- `find()` This gives the index of the first match it finds, or -1 if it can't find anything:

`SELECT find(BookExcerpt.excerpt, 'sofa');` produces `{-1, 151}` - the first `BookExcerpt.excerpt` doesn't have it, while the second has it at index 151.

- `split()` lets you make an array of the string split however you like. Most common is to split by `' '` to split by spaces. But this works too:

```
SELECT MinorVampire {
  names := (SELECT str_split(.name, 'n'))
};
```

Now the `n`s are all gone:

```
{
  default::MinorVampire {names: ['Woma', ' 1']},
  default::MinorVampire {names: ['Woma', ' 2']},
  default::MinorVampire {names: ['Woma', ' 3']},
  default::MinorVampire {names: ['Lucy Weste', 'ra']},
}
```

- `re_match()` for the first match and `re_match_all()` for all matches if you know how to use regular expressions and want to use those. This could be useful because the book Dracula was written over 100 years ago and has different spelling sometimes:

```
SELECT re_match_all('[Tt]o-?night', 'Dracula is an old book, so the word tonight is written to-night. Tonight we know how to write both tonight and to-night.');
{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}
```

`[Tt]o-?night` means words that: 
- start with a `T` or a `t`, 
- then `o`, 
- maybe have an `-` in between, 
- and end in `night`, 

so it gives: `{['tonight'], ['to-night']}`.


# Chapter 17 - Poor Renfield. Poor Mina.

> It turns out that Renfield was right. Dracula found out about the boxes and decided to attack Mina that night. He succeeds, and Mina is now slowly turning into a vampire (though she is still human). The group finds Renfield in a pool of blood, and dying. Renfield tells them that he thought Dracula would help him become a vampire too, but when Dracula ignored him and asked him to help get into Mina's room, he attacked Dracula to try to stop him from hurting Mina. Dracula, of course, was much stronger and won. Van Helsing does not give up though. If Mina is now connected to Dracula, what happens if he uses hypnotism on her? Could that work?

Remember the function `fight()` that we made? It was overloaded to take either `(Person, Person)` or `(str, Person)` as input. Let's give it Dracula and Renfield:

```
WITH 
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),
  renfield := (SELECT Person FILTER .name = 'Renfield'),
SELECT fight(dracula, renfield);
```

The output is of course `{'Count Dracula wins!'}`.

One other way to do the same query is with a single tuple, which we then put into the function:

```
WITH fighters := (
  (SELECT Person FILTER .name = 'Count Dracula'), (SELECT Person FILTER .name = 'Renfield')
  ),
  SELECT fight(fighters.0, fighters.1);
```

That's not bad, but if we wanted to, we could also give names to the items in the tuple instead of using `.0` and `.1`. It looks like this:

```
WITH fighters := (
  dracula := (SELECT Person FILTER .name = 'Count Dracula'), 
  renfield := (SELECT Person FILTER .name = 'Renfield')),
  SELECT fight(fighters.dracula, fighters.renfield);
```

Here's one more example of a named tuple:

```
WITH minor_vampires := (
  women := (SELECT MinorVampire FILTER .name LIKE '%Woman%'), 
  lucy := (SELECT MinorVampire FILTER .name LIKE '%Lucy%')),
SELECT (minor_vampires.women.name, minor_vampires.lucy.name);
```

The output is:

```
{('Woman 1', 'Lucy Westenra'), ('Woman 2', 'Lucy Westenra'), ('Woman 3', 'Lucy Westenra')}
```

Now we need to give Renfield a `last_appearance` with an update. Let's do a fancy one again where we can select the update we just made and display that information:

```
SELECT ( # Put the whole update inside
  UPDATE NPC filter .name = 'Renfield'
    SET {
  last_appearance := <cal::local_date>'1887-10-03'
}) # then use it to call up name and last_appearance
  {
  name, 
  last_appearance
  };
```

This gives us: `{default::NPC {name: 'Renfield', last_appearance: <cal::local_date>'1887-10-03'}}`

It would be cool right now to create a quick function to change the number of coffins in a place: we could write something like `change_coffins('London', -13)` to reduce it by 13, for example. But the problem right now is this: 

- the `HasCoffins` type is an abstract type, with one property: `coffins`
- places that can have coffins are `Place` and all the types from it, plus `Ship`,
- the best way to filter is by `.name`, but `HasCoffins` doesn't have this property.

So maybe we can turn this type into something else called `HasNameAndCoffins`. We can put the `name` and `coffins` properties inside there. This won't be a problem because in our game, every place needs a name and a number of coffins. Remember, 0 coffins means that vampires can't stay in a place for long: just quick trips in at night before the sun rises.

Here is the type with its new property. We'll give it two constraints: `exclusive` and `max_len_value` to keep names from being too long.

```
  abstract type HasNameAndCoffins {
    required property coffins -> int16 {
      default := 0;
  }
  required property name -> str {
      constraint exclusive;
      constraint max_len_value(30);
  }
}
```

So now we can change our `Ship` type (notice that we removed `name`)

```
    type Ship extending HasNameAndCoffins {
        multi link sailors -> Sailor;
        multi link crew -> Crewman;
    }
```

And the `Place` type. It's much simpler now.

```
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

Finally, we can change our `can_enter()` function. This one needed a `HasCoffins` type before: 

```
    function can_enter(person_name: str, place: HasCoffins) -> str
      using EdgeQL $$
        with vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
        SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
            $$;   
```

But now that `HasNameAndCoffins` holds `name`, we can just have the user enter a string. We'll change it to this:

```
function can_enter(person_name: str, place: str) -> str
  using EdgeQL $$
  with 
    vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
    place := (SELECT HasNameAndCoffins FILTER .name = place LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
    $$;   
```

And now we can just enter `can_enter('Count Dracula', 'Munich')` to get `'Count Dracula cannot enter.'` - Dracula didn't bring any coffins there.

Finally, we can make our `change_coffins` function. It's easy:

```
    function change_coffins(place_name: str, number: int16) -> HasNameAndCoffins
       using EdgeQL $$
         UPDATE HasNameAndCoffins FILTER .name = place_name
         SET {
           coffins := .coffins + number
         }
      $$
```

Now let's give the ship `The Demeter` some coffins.

```
SELECT change_coffins('The Demeter', <int16>10);
```

Then we'll make sure that it got them:

```
 SELECT Ship {
  name,
  coffins,
};
```

We get: `{default::Ship {name: 'The Demeter', coffins: 10}}`. The Demeter got its coffins!

# Chapter 18 - Using Dracula's own weapon against him

> Van Helsing was correct: Mina is connected to Dracula. He uses hypnotism to find out more about where he is and what he is doing. They find Dracula's other house in London with all his money. They know he will come to get it, and wait for him to arrive. All of a sudden Dracula runs into the house and attacks. Jonathan strikes with his knife, and cuts Dracula's bag with all his money. Dracula grabs some of the money that fell and jumps out the window, saying "You shall be sorry yet, each one of you! You think you have left me without a place to rest; but I have more. My revenge is just begun!" Then he disappears.

This is a good time to think about money in our game. The characters have been active in various countries like England, Romania and Germany, and each of those have their own money. We should create an `abstract type Currency` that we can use for all of these types of money.

Now, there is one difficulty here: in the 1800s, monetary systems were more complicated than they are today. In England, for example it wasn't 100 cents to 1 pound, it was 20 shillings to one pound, and 12 pence to one shilling. To reflect this, we'll say that `Currency` has three properties: `major`, `minor`, and `sub_minor`. Each one of these will have an amount, and finally there will be a number for the conversion, plus a `link owner -> Person`. So `Currency` will look like this:

```
    abstract type Currency {
        required link owner -> Person;
        required property major -> str;
        required property major_amount -> float64 {
            default := 0;
            constraint min_value(0);
        }
        required property minor -> str;
        required property minor_amount -> float64 {
            default := 0;
            constraint min_value(0);
        }
        required property minor_conversion -> int64;
        
        property sub_minor -> str;
        property sub_minor_amount -> float64 {
            default := 0;
            constraint min_value(0);
        }
        property sub_minor_conversion -> int64;
    }
```

We also gave it a constraint of `min_value(0)` so that characters won't be able to buy with money they don't have. We probably don't need to think about credit and other complicated things like that for the game.

Then comes our first currency: the `Pound` type. The `minor` property is called `'shilling'`, and we use `minor_conversion` to get the amount in pounds. The same thing happens with `'pence'`. Then our characters can collect various coins but the final value can still quickly be turned into pounds.

```
    type Pound extending Currency {
        overloaded required property major {
            default := 'pound'
        }
        overloaded required property minor { 
            default := 'shilling'
        }
        overloaded required property minor_conversion {
            default := 20
        }
        overloaded property sub_minor { 
            default := 'pence'
        }
        overloaded property sub_minor_conversion {
            default := 240
        }
    }
```

Now let's give Dracula some money. We'll give him 2500 pounds, 50 shillings, and 200 pence. Maybe that's a lot of money.

```
INSERT Pound {
 owner := (SELECT Person filter .name = 'Count Dracula'),
   major_amount := 2500,
   minor_amount := 50,
   sub_minor_amount := 200
};
```

Then we can use the conversion rates to display the total amount he owns in pounds:

```
SELECT Currency {
  owner: {name},
  total := .major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)
};
```

He has this many:

`{default::Pound {owner: default::Vampire {name: 'Count Dracula'}, total: 2503.3333333333335}}`

We know that Arthur has pretty much unlimited money, but the others we aren't sure about. Let's give them a random amount of money, and also `SELECT` it at the same time to display the result. For the random number we'll use the method we used for `strength` before: `round()` on a `random()` number multiplied by the maximum.

Finally, when displaying the total we will cast it to a `decimal` type. With this, we can display the number of pounds as something like 555.76 instead of 555.76545256. For this we use the same `round()` function, but the last one in these signatures:

```
std::round(value: int64) -> float64
std::round(value: float64) -> float64
std::round(value: bigint) -> bigint
std::round(value: decimal) -> decimal
std::round(value: decimal, d: int64) -> decimal
```

The `d: int64` part is the number of decimal places we want to give it.

All together, it looks like this:

```
SELECT (FOR character IN {'Jonathan Harker', 'Mina Murray', 'The innkeeper', 'Emil Sinclair'}
  UNION (
    INSERT Pound {
      owner := (SELECT Person FILTER .name = character LIMIT 1),
      major_amount := (SELECT(round(random() * 500))),
      minor_amount := (SELECT(round(random() * 100))),
      sub_minor_amount := (SELECT(round(random() * 500)))
  })) {
 owner: {
   name
  },
 pounds := .major_amount,
 shillings := .minor_amount,
 pence := .sub_minor_amount,
 total_pounds := (SELECT
    (round(<decimal>(.major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)), 2)))
 };
```

And then it will give a result similar to this with our collections of money, each with an owner:

```
{
  default::Pound {owner: default::NPC {name: 'Jonathan Harker'}, pounds: 54, shillings: 100, pence: 256, total_pounds: 60.07n},
  default::Pound {owner: default::NPC {name: 'Mina Murray'}, pounds: 360, shillings: 77, pence: 397, total_pounds: 365.50n},
  default::Pound {owner: default::NPC {name: 'The innkeeper'}, pounds: 87, shillings: 36, pence: 23, total_pounds: 88.90n},
  default::Pound {owner: default::PC {name: 'Emil Sinclair'}, pounds: 427, shillings: 19, pence: 88, total_pounds: 428.32n},
}
```

We are nearing the end of the book, and should probably start to clean up the schema and inserts a bit.

First, we have two inserts here where we could only have one.

```
INSERT City {
  name := 'Munich',
};

INSERT City {
    name := 'London',
};
```

We'll change that to an insert with a `UNION`:

```
 FOR city_name IN {'Munich', 'London'}
   UNION (
   INSERT City {
     name := city_name
    }
  );
```

Then we'll do the same for the `Country` types with the names Romania and Slovakia. Now they are a single insert:

```
FOR country_name IN {'Romania', 'Slovakia'}
  UNION (
    INSERT Country {
      name := country_name
    }
  );
```

The other `City` inserts are a bit different: some have `modern_name` and others have `population`. In a real game we would insert them all in this sort of form, all at once:

```
FOR city IN {
  ('City 1', 'Modern City 1', 800),
  ('City 2', 'Modern City 2', 900),
  ('City 3', 'Modern City 3', 455),
 }
  UNION (
    INSERT City {
      name := city.0,
      modern_name := city.1,
      population := city.2
});
```

And the same would take place with all the `NPC` types with `first_appearance` and so on. But we don't have that many cities and characters to insert in this tutorial so we don't need to be so systematic yet.

We can also turn the inserts for the `Ship` type into a single one. Right now it looks like this:

```
FOR n IN {1, 2, 3, 4, 5}
  UNION (
  INSERT Crewman {
  number := n,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
});

INSERT Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

INSERT Sailor {
  name := 'The First Mate',
  rank := 'First mate'
};

INSERT Sailor {
  name := 'The Second Mate',
  rank := 'Second mate'
};

INSERT Sailor {
  name := 'The Cook',
  rank := 'Cook'
};

INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

Let's put that all together:

```
INSERT Ship {
   name := 'The Demeter',
   sailors := {
    (INSERT Sailor {
       name := 'The Captain',
       rank := 'Captain'
     }),
    (INSERT Sailor {
       name := 'The First Mate',
       rank := 'First mate'
     }),
    (INSERT Sailor {
       name := 'The Second Mate',
       rank := 'Second mate'
     }),
    (INSERT Sailor {
       name := 'The Cook',
       rank := 'Cook'
     })
 },
   crew := (
 FOR n IN {1, 2, 3, 4, 5}
   UNION (
   INSERT Crewman {
   number := n,
   first_appearance := cal::to_local_date(1887, 7, 6),
   last_appearance := cal::to_local_date(1887, 7, 16),
   })
 )
};
```

Much better!

# Chapter 19 - Dracula flees back to Transylvania

> Thanks to Mina, they know that Dracula has fled on a ship with his last box and is going back to Transylvania. One team (Van Helsing and Mina) goes to Castle Dracula, while the others go to Varna to try to catch the ship when it arrives. Jonathan Harker just sharpens his knife, and looks very scary now - he wants to kill Dracula as soon as possible and save his wife. But where is the ship? Every day they wait, and wait...and then one day, they get a telegram that says that the ship arrived at Galatz, not Varna. Is it too late? They rush off up the river to try to find Dracula.

We have another ship moving around the map in this chapter. Last time, we made a `Ship` type that looks like this:

```
type Ship extending HasNameAndCoffins {
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

That's not bad, but it doesn't have any information about visits made by which ship to where. Let's make a quick type that contains all the information on ship visits. Each visit will have a link to a `Ship` and a `Place`, and a `cal::local_date`. It looks like this:

```
type Visit {
  link ship -> Ship;
  link place -> Place;
  property date -> cal::local_date;
}
```

This new ship that Dracula is on is called the `Czarina Catherine`. Let's use that to insert a few visits from the ships we know.

```
FOR visit in {
    ('The Demeter', 'Varna', '1887-07-06'),
    ('The Demeter', 'Bosphorus', '1887-07-11'),
    ('The Demeter', 'Whitby', '1887-08-08'),
    ('Czarina Catherine', 'London', '1887-10-05'),
    ('Czarina Catherine', 'Galatz', '1887-10-28')}
    UNION (
 INSERT Visit {
   ship := (SELECT Ship FILTER .name = visit.0),
   place := (SELECT Place FILTER .name = visit.1),
   date := <cal::local_date>visit.2
   });
```



# Chapter 20 - The final battle

> Mina is almost a vampire now, and says she can feel Dracula all the time. Van Helsing arrives at Castle Dracula and Mina waits outside. Van Helsing then goes inside and destroys the vampire women. Meanwhile, the other men approach from the south and are also close to Castle Dracula. They find a group of friends of Dracula who have him inside his box, carrying him on a wagon. The sun is almost down, it is snowing, and they need to hurry. They get closer and closer, and grab the box. They pull the nails back and open it up, and see Dracula lying inside. Jonathan pulls out his knife. But just then the sun goes down. Dracula opens his eyes with a look of triumph, and...

This is almost the end of the book, but we won't spoil the final ending. If you want to read it now, just [check out the book on Gutenberg](http://www.gutenberg.org/files/345/345-h/345-h.htm#CHAPTER_XIX) and search for "the look of hate in them turned to triumph".
