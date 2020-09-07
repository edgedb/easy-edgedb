# Welcome!

Welcome to the tutorial to easily learn EdgeDB in plain English.

In this course, we will be using EdgeDB and its query language EdgeQL to put together a database. We will imagine that we are creating the database for a game that is based on the book Bram Stoker's Dracula. Because the game is going to use the events in the book, we need a database to tie everything together. It will need to show the connections between the characters, locations, dates, and more.

# Chapter 1 - Jonathan Harker heads towards Transylvania

In the beginning of the book, the main character Jonathan Harker is a young lawyer who is going to meet a client. The client is a nobleman named Count Dracula who lives somewhere in Eastern Europe. Here is how the book begins:

>3 May. Bistritz.—Left Munich at 8:35 P.M., on 1st May, arriving at Vienna early next morning; should have arrived at 6:46, but train was an hour late. Buda-Pesth seems a wonderful place, from the glimpse which I got of it from the train...

This is already a lot of information, and it helps us start to think about our database schema. EdgeQL uses something called [SDL (schema definition language)](https://edgedb.com/docs/edgeql/sdl/index#ref-eql-sdl) that makes migration easy. So far our schema needs the following:

- Some kind of City or Location type. Cities should have a name and a location, and sometimes a different name or spelling. Bistritz for example in the book is now called Bistrița, and Buda-Pesth is now written Budapest.
- Some kind of Person type. We need it to have a name, and also a way to track the places that the person visited.
 
To make a type in EdgeQL, just use the keyword `type` followed by the type name, then `{}` curly brackets. Our `Person` type will start out like this:

```
type Person {
}
```

Inside this we insert the properties for our `Person` type. Type `required property` if the type needs it, and just `property` if it is optional. 

```
type Person {
  required property name -> str;
  property places_visited -> array<str>;
}
```

A `str` is just a string, and is written inside either single quotes: `'Jonathan Harker'` or double quotes: `"Jonathan Harker"`. An `array` is a collection type, and our array here is an array of `str`s. We want it to look like this: `["Bistritz", "Vienna", "Buda-Pesth"]`. With that we can easily search later and see which character has visited where.

