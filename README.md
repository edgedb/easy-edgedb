# Welcome!

Welcome to the tutorial to easily learn EdgeDB in plain English. "Plain English" means using simple phrases and shorter sentences so you can quickly get the information you need. This lets you learn EdgeDB with a minimum of distractions, and is helpful if English is your second (or third, fourth...) language.

Nevertheless, simple English doesn't mean boring. We will imagine that we are creating the database for a game that is based on the book Bram Stoker's Dracula. Because it will use the events in the book for the background, we need a database to tie everything together. It will need to show the connections between the characters, locations, dates, and more.


# Chapter 1 - Jonathan Harker travels to Transylvania

![thththt](Sample_image.png)

In the beginning of the book we see the main character Jonathan Harker, who is a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan still doesn't know that Count Dracula is a vampire, so he's enjoying the trip to a new part of Europe. Here is how the book begins, with parts that are good for a database in **bold**:

>**3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

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

`places_visited` is not a `required` property because we might start adding people that we don't care about. Maybe one person will be the "innkeeper_in_bistritz" or something, and we won't care about `places_visited` for him.

Now for our City type:

```
type City {
  required property name -> str;
  property modern_name -> str;
}
```

This is similar: all cities have a name in the book, but some don't have a different modern name. Vienna is still Vienna, for example. We are imagining that our game will link the city names to their modern names so we can easily place them on a map.

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
  places_visited
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

So we can add it by using SDL when we do the migration. On top of SDL, EdgeQL also has something called [DDL (data definition language)](https://edgedb.com/docs/edgeql/ddl/index) that is more low level. DDL is good for quick changes, but not as good for large migrations. One reason why is that the order matters in DDL, but not in SDL. That means that if you have a type `City` that is connected to a type `Country`, you need to create `Country` first in DDL. In SDL, the other doesn't matter. 

But here we will look at a quick example of DDL. If we've already started our database, we can quickly change it like this:

```
ALTER TYPE City {
  CREATE PROPERTY important places -> array<str>
};
```

And it will say `OK: ALTER` to tell us that it worked. For most of this course we will keep using SDL, so when we change a type we are changing the migration structure. But you can remember that DDL is also an option for quick changes when you need them.

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

# Chapter 3 - Jonathan goes to Castle Dracula

In this chapter we are going to start to think about time, as you can see from what Jonathan Harker is doing:

>Jonathan Harker has just arrived at Castle Dracula after a ride in the carriage through the mountains. The ride was terrible: there was snow, strange blue fires and wolves everywhere. It was night when he arrived, and he meets and talks with Count Dracula. Dracula leaves before the sun rises though, because vampires are hurt by sunlight. Jonathan doesn't know that he's a vampire yet.

This is a good time to create a `Vampire` type. We can extend it from `abstract type Person` because that type only has `name` and `places_visited`, which are good to have for `Vampire` too. But vampires are different from humans because they can live forever. Let's add `age` to `Person` so that all the other types can use it too. Now `Person' looks like this:

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

and then create Count Dracula. We know that he lives in Romania, but that isn't a city. This is a good time to change the `City` type. We'll change the name to `Place` and make it an `abstract type`, and then `City` can extend from it. We'll also add a `Country` type that does the same thing. Now they look like this:

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

# Chapter 5 - Jonathan tries to leave the castle

Here's what happens in this chapter:

>During the day, Jonathan decides to try to explore the castle but too many doors and windows are locked. He doesn't know how to get out, and wishes he could at least send Mina a letter. He pretends that there is no problem, and keeps talking to Dracula during the night. One night he sees Dracula climb out of his window and down the castle wall, like a snake, and now he is very afraid. A few days later he breaks one of the doors and finds another part of the castle. The room is very strange and he feels sleepy. He opens his eyes and sees three vampire women next to him. He can't move.

Since Jonathan was thinking of Mina back in London, let's learn about `std::datetime` because it uses time zones. To create a datetime, you can just cast a string in ISO 8601 format with `<datetime>`. That format looks like this:

`'2020-12-06T22:12:10Z'`

YYYY-MM-DDTHH:MM:SSZ

The `T` inside there is just a separator, and the `Z` at the end means "zero timeline". That means that it is 0 different (offset) from UTC: in other words, it *is* UTC.

One other way to get a `datetime` is to use the `to_datetime()` function. [Here is its signature](https://edgedb.com/docs/edgeql/funcops/datetime/#function::std::to_datetime), which shows that there are multiple ways to make a `datetime` with this function. The easiest is probably the third, which looks like this: `std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime`

With this, in our game we could have a function that just generates integers for times that then use `to_datetime` for a proper time stamp. Let's imagine that it's 10:35 am in Castle Dracula and Jonathan is trying to escape on May 12 to send Mina a letter. In Romania the time zone is 'EEST' (Eastern European Summer Time). We'll use `to_datetime()` to generate this. We won't worry about the year, because the story of Dracula takes place in the same year - we'll just use 2020 for convenience. We type this:

`SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');`

And get the following output:

`{<datetime>'2020-05-12T07:35:00Z'}`

The `07:35:00` part shows that it was automatically converted to UTC, which is London where Mina lives.

With this, we can see the duration between events, because EdgeDB has a `duration` type that you get when subtracting a datetime from another one. Let's see the exact number of seconds between one date in Central Europe and another in Korea:

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
