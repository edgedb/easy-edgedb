# Welcome!

Welcome to the tutorial to easily learn EdgeDB in plain English. "Plain English" means using simple phrases and shorter sentences so you can quickly get the information you need. It's ideal if English is your second language, or if you just prefer to get the information quickly.

Nevertheless, simple English doesn't mean boring. We will imagine that we are creating the database for a game that is based on the book Bram Stoker's Dracula. Because it will use the events in the book for the background, we need a database to tie everything together. It will need to show the connections between the characters, locations, dates, and more.


# Chapter 1 - Jonathan Harker heads towards Transylvania

In the beginning of the book, we see the main character Jonathan Harker who a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan still doesn't know that Count Dracula is a vampire, so he's enjoying the trip to a new part of Europe. Here is how the book begins, with parts that could be good for a database in bold:

>**3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

This is already a lot of information, and it helps us start to think about our database schema. EdgeQL uses something called [SDL (schema definition language)](https://edgedb.com/docs/edgeql/sdl/index#ref-eql-sdl) that makes migration easy. So far our schema needs the following:

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

It is also what shows up when you use `SELECT` to select a type. Just typing `SELECT` with a type will show you all the `uuid`s for the type. Let's look at all the cities we have so far:

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
....... modern_name,
....... };
```

You will remember that one of our cities doesn't have a modern name. It still shows up though as an "empty set", because every value in EdgeDB is a set of elements. Here is the result:

```
{Object {modern_name: {}}, Object {modern_name: 'Budapest'}, Object {modern_name: 'Bistrița'}}
```

So there is some object with an empty set for `modern_name`, while the other two have a name. This shows us that EdgeDB doesn't have `null` like in some languages: if nothing is there, it will return an empty set.

# Chapter 2 - at the Golden Krone Hotel

We continue to read the story as we think about the database we need to store the information. The important information is in bold.

In Chapter 2, Jonathan Harker has found a hotel in **Bistritz**, called the **Golden Krone Hotel**. He gets a welcome letter there from Dracula, who is waiting in his **castle**. Jonathan Harker will have to take a **horse-driven carriage** to get there tomorrow. We also see that Jonathan Harker is from **London**. The innkeeper at the Golden Krone Hotel seems very afraid of Dracula. He doesn't want Jonathan to leave and says it will be dangerous, but Jonathan doesn't listen.

