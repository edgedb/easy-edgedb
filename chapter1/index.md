---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# Chapter 1 - Jonathan Harker travels to Transylvania

In the beginning of the book we see the main character Jonathan Harker, a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan is enjoying the trip to a new part of Europe, completely unaware that Count Dracula is actually a vampire. The book begins with Jonathan writing in his journal as he travels. The parts to note when thinking about a database are in **bold**:

> **3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

## Schema, object types

This is already a lot of information, and it helps us start to think about our database schema. The language used for EdgeDB is called EdgeQL, and is used to define, mutate, and query data. Inside it is {ref}`SDL (schema definition language)<docs:ref_eql_sdl>` that makes migration easy, and which we will learn in this book. So far our schema needs the following:

- Some kind of `City` or `Location` type. These types that we can create are called {ref}`object types <docs:ref_datamodel_object_types>`, made out of properties and links. What properties should a City type have? Perhaps a name and a location, and sometimes a different name or spelling. Bistritz for example is in Romania and is now written Bistrița (note the ț - it's Bistrița, not Bistrita), while Buda-Pesth is now written Budapest.
- Some kind of `Person` type. We need it to have a name, and also a way to track the places that the person visited.

To make a type inside a schema, just use the keyword `type` followed by the type name, then `{}` curly brackets. Our `Person` type will start out like this:

```sdl
type Person {
}
```

That's all you need to create a type.

But hold on, where is our schema? The best way to create a schema is to start an EdgeDB project. It is quite easy.

* First make sure you have [EdgeDB installed on your computer](https://www.edgedb.com/install). You will now have the EdgeDB CLI installed which allows you to make new projects and apply migrations (among other things).
* Once this is done, make an empty directory where you want to hold your project. Now open up a command line. If you are on Windows and haven't opened a command line before, type the Windows key and `cmd`, then click on Command Prompt. Then type `cd c:/easy-edgedb` or whatever the name of your directory is.
* Type `edgedb project init`. The EdgeDB CLI will now ask you a few questions. The output should look something like this:

```
No `edgedb.toml` found in `\\?\C:\easy-edgedb` or above
Do you want to initialize a new project? [Y/n]
> Y
Specify the name of EdgeDB instance to use with this project [default: easy_edgedb]:
> easy_edgedb
Checking EdgeDB versions...
Specify the version of EdgeDB to use with this project [default: 2.14]:
> 2.14
┌─────────────────────┬─────────────────────────────────────┐
│ Project directory   │ \\?\C:\easy-edgedb             │
│ Project config      │ \\?\C:\easy-edgedb\edgedb.toml │
│ Schema dir (empty)  │ \\?\C:\easy-edgedb\dbschema    │
│ Installation method │ WSL                                 │
│ Version             │ 2.14+7aec755                        │
│ Instance name       │ easy                                │
└─────────────────────┴─────────────────────────────────────┘
Version 2.14+7aec755 is already downloaded
Initializing EdgeDB instance...
Applying migrations...
Everything is up to date. Revision initial
Project initialized.
To connect to easy_edgedb, run `edgedb`
```

That was easy! Now try typing `edgedb`. This will take you into the EdgeDB REPL. Now type `select 'Jonathan Harker';` and hit enter. You should see the following output:

```
edgedb> select "Jonathan Harker";
{'Jonathan Harker'}
edgedb>
```

You just made your first query in EdgeDB. Now type `\quit` to escape the REPL and get back to the command line.

So now let's take a look at the schema so we can create our `Person` type. The CLI has already let us know that the schema is inside the `\edgedb\dbschema` directory, so go inside there and look for a file called `default.esdl`. This is your schema. At the moment, all it holds is a module called `default`:

```sdl
module default {

}
```

This module is where we will hold the types for our database, including our `Person` type. Let's give it a try! Add the `Person` type that we made above, as follows:

```sdl
module default {
  type Person {
  }
}
```

Make sure the file is saved, then type `edgedb migration create`. You should see some output that looks like the following:

```
Created c:\easy-edgedb\dbschema\migrations\00001.edgeql, id: m14c35zu7lzg46r2337nwehaihkv6d4xwcatu6ulogd5kqafjtrnra
```

Now type `edgedb migrate`. The CLI will apply the migration with the following output:

```
Applied m14c35zu7lzg46r2337nwehaihkv6d4xwcatu6ulogd5kqafjtrnra (00001.edgeql)
```

And you are done!

To sum up, an EdgeDB migration simply consists of the following:

* Changing your schema,
* Typing `edgedb migration create`,
* Typing `edgedb migrate`.

We will be doing a lot of that in this book! Even by the second chapter you will be very used to this process.

```{eval-rst}
.. note::
  The CLI creates a new file upon each migration to generate the commands to change the schema to the one we want. The first file will be called 00001.edgeql, the second will be 00002.edgeql, and so on. These files are quite readable so feel free to take a look at if you are curious. But note that they use a syntax called DDL (Data Definition Language) that gives commands to EdgeDB one at a time, and you do not need to learn it. Human users of EdgeDB use a language called SDL (Schema Definition Language) that simply declares what a schema will look like. The CLI then automatically creates DDL commands to make it happen.
```

Now that we know how to do a schema migration, let's add some properties to our `Person` type. Use `required property` if the type needs it, and just `property` if it is optional. Let's give the `Person` type a name and an array (a collection) of places visited:

```sdl
type Person {
  required property name -> str;
  property places_visited -> array<str>;
}
```

With `required property name` our `Person` objects are always guaranteed to have a name - you can't make a `Person` object without it. Here's the error message if you try:

```
MissingRequiredError: missing value for required property default::Person.name
```

A `str` is just a string, and goes inside either single quotes: `'Jonathan Harker'` or double quotes: `"Jonathan Harker"`. The `\` escape character before a quote makes EdgeDB treat it like just another letter: `'Jonathan Harker\'s journal'`.

An `array` is a collection of the same type, and our array here is an array of `str`s. We want it to look like this: `["Bistritz", "Munich", "Buda-Pesth"]`. The idea is to easily search later and see which character has visited where.

`places_visited` is not a `required` property because we might later add minor characters that don't go anywhere. Maybe one person will be the "innkeeper_in_bistritz" or something, and we won't know or care about `places_visited` for him.

Now for our City type:

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
}
```

This is similar, just properties with strings. The book Dracula was published in 1897 when spelling for cities was sometimes different. All cities have a name in the book (that's why it's `required`), but some won't need a different modern name. Munich is still Munich, for example. We are imagining that our game will link the city names to their modern names so we can easily place them on a map.

## Migration

With our two new types added, it's time for a migration! You know what that means: `edgedb migration create` followed by `edgedb migrate`.

Interestingly though, this time the CLI is asking us a few questions about the changes we made. Here is the first question.

```
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
>
```

The CLI does this to make sure that it is applying the right changes to the schema, and to give us the opportunity to make changes if it has misunderstood anything. Most of the time you will just end up clicking `y` over and over again for each change. But there are other possible answers we can give. Type `?` if you want to see those.

One of those is `l`, which means:

```
l or list - list the DDL statements associated with prompt
```

Sure, let's be curious and try typing `l` to see what commands will be generated to create our `City` type. If we type `l` we will see the following:

```edgeql
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
> l
The following DDL statements will be applied:
    CREATE TYPE default::City {
        CREATE PROPERTY modern_name -> std::str;
        CREATE REQUIRED PROPERTY name -> std::str;
    };
```

Looks good! It's a type `City` inside the module `default`, it has a `required property name` and a `property modern_name`. Let's now just type `y` for everything. After typing `y` two times, the CLI asks us a sudden question:

```
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
> y
Did you alter object type 'default::Person'? [y,n,l,c,b,s,q,?]
> y
Please specify an expression to populate existing objects in order to make property 'name' of object type 'default::Person' required:
fill_expr>
```

The CLI is essentially saying: "There might be `Person` objects in the database already. But now they all need to have a `name` property, which wasn't required before. How should I decide what `name` to give them?"

Fortunately, the expression here is pretty simple: let's just give them all an empty string. Type `''` and hit enter, and the CLI will now be happy with the migration. Don't forget to complete the migration with `edgedb migration`, and we are done!

There are a lot of other commands beyond the commands for migration, though we won't need them for this book. You could bookmark these four pages for later use, however:

- {ref}`Admin commands <docs:ref_cheatsheet_admin>`: Creating user roles, setting passwords, configuring ports, etc.
- {ref}`CLI commands <docs:ref_cheatsheet_cli>`: Creating databases, roles, setting passwords for roles, connecting to databases, etc.
- {ref}`REPL commands <docs:ref_cheatsheet_repl>`: Mostly shortcuts for a lot of the commands we'll be using in this book.
- {ref}`Various commands <docs:ref_eql_statements_rollback_tx>` about rolling back transactions, declaring savepoints, and so on.

There are also a few places to download packages to highlight your syntax if you like. EdgeDB has these packages available for [Atom](https://atom.io/packages/edgedb), [Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb), [Sublime Text](https://packagecontrol.io/packages/EdgeDB), and [Vim](https://github.com/edgedb/edgedb-vim).

Now let's start playing with some data!

## Selecting

The `select` keyword is the main query command in EdgeDB, and you use it to see results based on the input that comes after it. Keywords in EdgeDB are case insensitive, so `select`, `SELECT` and `SeLeCT` are all the same.

Let's give `select` a try with something really easy: just selecting a string. Type `edgedb` to log into the REPL and give this a try:

```edgeql
select 'Jonathan Harker begins his journey.';
```

This returns `{'Jonathan Harker begins his journey.'}`, no surprise there. Did you notice that it's returned inside a `{}`? The `{}` means that it's a set, and in fact {ref}`everything in EdgeDB is a set <docs:ref_eql_everything_is_a_set>` (make sure to remember that). It's also why EdgeDB doesn't have null: where you would have null in other languages, EdgeDB just gives you an empty set: `{}`. The advantage here is that sets always behave the same way even when they are empty, instead of [all the unwelcome surprises](https://www.edgedb.com/blog/we-can-do-better-than-sql#null-a-bag-of-surprises) that come with using null.

For the next `select` queries, we will use some more operators that use the `=` sign:

- `:=` is used to declare,
- `=` is used to check equality (not `==`),
- `!=` is the opposite of `=`.

Let's use `:=` to assign a variable:

```edgeql
select jonathans_name := 'Jonathan Harker';
```

This just returns what we gave it: `{'Jonathan Harker'}`. But this time it's a string that we assigned called `jonathans_name` that is being returned.

Now let's do something with this variable. We can use the keyword `select` to use this variable and then compare it to `'Count Dracula'`:

```edgeql
select jonathans_name := 'Jonathan Harker',
select jonathans_name != 'Count Dracula';
```

The output is `{true}`. Of course, you can just write `select 'Jonathan Harker' != 'Count Dracula'` for the same result. Soon we will do more complex operations with the variables that we assign with `:=`.

## Inserting objects

Let's start inserting some objects with the schema we already have. Later on we can think about adding time zones and locations for the cities for our imaginary game. But in the meantime, we will add some items to the database using `insert`.

Don't forget to separate each property by a comma, and finish the `insert` with a semicolon. Indentation isn't relevant like it is in languages such as Python and F#, but EdgeDB prefers two spaces for indentation.

```edgeql
insert City {
  name := 'Munich',
};

insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};
```

Note that a comma after the last item is optional - you can put it in or leave it out. Here we put a comma at the end sometimes and left it out at other times to show this.

Finally, the `Person` insert would look like this, but don't insert it yet:

```edgeql
insert Person {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Munich", "Buda-Pesth"],
};
```

Because hold on a second...that insert won't link a `Person` object it to any of the `City` inserts that we already did. Here's where our schema needs some improvement:

- We have a `Person` type and a `City` type,
- The `Person` type has the property `places_visited` with the names of the cities, but they are just strings in an array. It would be better to link this property to the `City` type somehow.

So let's not do that `Person` insert. We'll fix the `Person` type soon by changing `array<str>` from a `property` to a `multi link` to the `City` type. This will actually join them together.

But first let's look a bit closer at what happens when we use `insert`.

As you can see, strings (`str`) are fine with unicode letters like ț. Even emojis and special characters are just fine, so you could even create a `City` called '🤠' or '(╯°□°)╯︵ ┻━┻' if you wanted to.

EdgeDB also has a byte literal type that gives you the bytes of a string. This is mainly for raw data that humans don't need to view such when saving to files. They must be characters that are 1 byte long.

You create byte literals by adding a `b` in front of the string:

```edgeql-repl
edgedb> select b'Bistritz';
{b'Bistritz'}
```

And because the characters must be 1 byte, only ASCII works for this type. So the name in `modern_name` as a byte literal will generate an error because of the `ț`:

```edgeql-repl
edgedb> select b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
  ┌─ query:1:8
  │
1 │ select b'Bistrița';
  │        ^ error
```

Every time you `insert` an item, EdgeDB gives you a `uuid` back. UUID stands for [Universally Unique IDentifier](https://en.wikipedia.org/wiki/Universally_unique_identifier), and is used widely to make identifiers that won't be used anywhere else. It will look like this:

```
{default::Person {id: 462b29ea-ff3d-11eb-aeb7-b3cf3ba28fb9}}
```

You probably noticed this already when we inserted our first three `City` objects.

It is also what shows up when you use `select` to select a type. Just typing `select` with a type will show you all the `uuid`s for the type. Let's look at all the cities we have so far:

```edgeql
select City;
```

This gives us three items:

```
{
  default::City {id: 4ba1074e-ff3f-11eb-aeb7-cf15feb714ef},
  default::City {id: 4bab8188-ff3f-11eb-aeb7-f7b506bd047e},
  default::City {id: 4bacf860-ff3f-11eb-aeb7-97025b4d95af},
}
```

This only tells us that there are three objects of type `City`. To see inside them, we can add property or link names to the query. This is called describing the {ref}`shape <docs:ref_eql_shapes>` of the data we want. We'll select all `City` types and display their `modern_name` with this query:

```edgeql
select City {
  modern_name,
};
```

Once again, you don't need the comma after `modern_name` because it's at the end of the query.

You will remember that one of our cities (Munich) doesn't have anything for `modern_name`. But it still shows up as an "empty set", because every value in EdgeDB is a set of elements, even if there's nothing inside. Here is the result:

```
{
  default::City {modern_name: {}},
  default::City {modern_name: 'Budapest'},
  default::City {modern_name: 'Bistrița'},
}
```

So there is some object with an empty set for `modern_name`, while the other two have a name. This shows us that EdgeDB doesn't have `null` like in some languages: if nothing is there, it will return an empty set.

The first object is a mystery so we'll add `name` to the query so we can see that it's the city of Munich:

```edgeql
select City {
  name,
  modern_name
};
```

This gives the output:

```
{
  default::City {name: 'Munich', modern_name: {}},
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

If you just want to return a single property or link of an object, you can use `.` after the type name. For example, `select City.modern_name` will return a set of strings for all of the `City` objects in the database so far. It will give this output:

```
{'Budapest', 'Bistrița'}
```

This type of expression is called a _path expression_ or a _path_, because it is the direct path to the values inside. And each `.` moves on to the next path, if there is another one to follow.

You can also change property names like `modern_name` to any other name if you want by using `:=` after the name you want. Those names you choose become the variable names that are displayed. For example:

```edgeql
select City {
  name_in_dracula := .name,
  name_today := .modern_name,
};
```

This prints:

```
{
  default::City {name_in_dracula: 'Munich', name_today: {}},
  default::City {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest'},
  default::City {name_in_dracula: 'Bistritz', name_today: 'Bistrița'},
}
```

This will not change anything inside the schema - it's just a quick variable name to use in a query.

By the way, `.name` is short for `City.name`. You can also write `City.name` each time (that's called the _fully qualified name_), but it's not required.

So if you can make a quick `name_in_dracula` property from `.name`, can we make other things too? Indeed we can. For the moment we'll just keep it simple but here is one example:

```edgeql
select City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'This is a city in the book Dracula'
};
```

And here is the output:

```
{
  default::City {
    name_in_dracula: 'Munich',
    name_today: {},
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Buda-Pesth',
    name_today: 'Budapest',
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Bistritz',
    name_today: 'Bistrița',
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
}
```

This brings up an interesting discussion about type safety. EdgeDB is strongly typed, meaning that everything needs a type and it will not try to mix different types together. So this query will not work: 

```edgeql
edgedb> select City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'This is a city in the book Dracula written in the year' + 1897
 };
```

EdgeDB refuses with the message `operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'`. But we didn't declare a type for `oh_and_by_the_way`, so how did EdgeDB know that it was a `str`?

EdgeDB uses what is known as "type inference" to guess the type, meaning that it can usually figure out the type itself. That is what happens here: EdgeDB knows that we are creating a `str` because we enclosed it in quotes. In other words, the type is still a concrete `str` even if we didn't specify that it was.

In fact, we can prove that these types are concrete right away: let's just ask EdgeDB.

```edgeql
select 'Jonathan Harker' is str;
```

This returns `{true}`. Let's try another:

```edgeql
select 8 is int64;
```

This returns `{true}`. Let's try one more!

```edgeql
select 8 is int32;
```

This time it returns `{false}`. As you can see, EdgeDB is selecting a concrete type every time even if we don't specify a type.

There is a way to change one type to another that we will learn in the next chapter.

## Links

So now the last thing left to do is to change our `property` in `Person` called `places_visited` to a `link`. Right now, `places_visited` gives us the names we want, but it makes more sense to link `Person` and `City` together. After all, the `City` type has `.name` inside it which is better to link to than rewriting everything inside `Person`. We'll change `Person` to look like this:

```sdl
type Person {
  required property name -> str;
  multi link places_visited -> City;
}
```

We wrote `multi` in front of `link` because one `Person` should be able to link to more than one `City`. The opposite of `multi` is `single`, which only allows one object to link to it. But `single` is the default, so if you just write `link` then EdgeDB will treat it as `single`.

Now when we insert Jonathan Harker, he will be connected to the type `City`. Don't forget that `places_visited` is not `required`, so we can still insert with just his name to create him:

```edgeql
insert Person {
  name := 'Jonathan Harker',
};
```

But this would only create a `Person` type connected to the `City` type but with nothing in it. Let's see what's inside:

```edgeql
select Person {
  name,
  places_visited
};
```

Here is the output: `{default::Person {name: 'Jonathan Harker', places_visited: {}}}`

But we want to have Jonathan be connected to the cities he has traveled to. We'll change `places_visited` when we `insert` to `places_visited := City`:

```edgeql
insert Person {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

Now let's see the places that Jonathan has visited. The code below is almost but not quite what we need:

```edgeql
select Person {
  name,
  places_visited
};
```

Here is the output:

```
{
  default::Person {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {id: 4ba1074e-ff3f-11eb-aeb7-cf15feb714ef},
      default::City {id: 4bab8188-ff3f-11eb-aeb7-f7b506bd047e},
      default::City {id: 4bacf860-ff3f-11eb-aeb7-97025b4d95af},
    },
  },
}
```

Close! But we didn't mention any properties inside `City` so we just got the object id numbers. Now we just need to let EdgeDB know that we want to see the `name` property of the `City` type. To do that, add a colon and then put `name` inside curly brackets.

```edgeql
select Person {
  name,
  places_visited: {
    name
  }
};
```

Success! Now we get the output we wanted:

```
{
  default::Person {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
    },
  },
}
```

Interestingly, Jonathan Harker has been inserted with a link to every city in the database. In other words, `places_visited := City` means `places_visited := every City type in the database`. Right now we only have three `City` objects, so this is no problem yet. But later on we will have more cities and won't be able to just write `places_visited := City` for all the other characters. For that we will need `filter`, which we will learn to use in the next chapter.

Note that if you have inserted "Johnathan Harker" multiple times, you will have multiple `Person` objects with that name corresponding to each `insert` command. This is OK for now. In [Chapter 7](../chapter7/index.md) we will learn how to make sure the database doesn't allow multiple copies of `Person` with the same name.

[Here is all our code so far up to Chapter 1.](code.md)

<!-- quiz-start -->

## Time to practice

1. Entering the code below returns an error. Try adding one character to make it return `{true}`.

   ```edgeql
   with my_name = 'Timothy',
   select my_name != 'Benjamin';
   ```

2. Try inserting a `City` called Constantinople, but now known as İstanbul.
3. Try displaying all the names of the cities in the database. (Hint: you can do it in a single line of code and won't need `{}` to do it)
4. Try selecting all the `City` types along with their `name` and `modern_name` properties, but change `.name` to say `old_name` and change `modern_name` to say `name_now`.
5. Will typing `SelecT City;` produce an error?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan Harker arrives in Romania._
