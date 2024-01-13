---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# Chapter 1 - Jonathan Harker travels to Transylvania

In the beginning of the book we see the main character Jonathan Harker, a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan is enjoying the trip to a new part of Europe, completely unaware that Count Dracula is actually a vampire. The book begins with Jonathan writing in his journal as he travels. The parts in his journal to note when thinking about a database are in **bold**:

> **3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

## Schema, object types

The language used for EdgeDB is called EdgeQL, and is used to define, mutate, and query data. The schema in your database uses {ref}`SDL (schema definition language)<docs:ref_eql_sdl>` that makes migration easy, and which we will learn in this book. The quote from Jonathan Harker's journal can help us start to think about our database. We'll start with our schema, which needs the following:

- Some kind of `City` or `Location` type. These types that we can create are called {ref}`object types <docs:ref_datamodel_object_types>`, made out of properties and links to other objects. What properties should a `City` type have? Perhaps a name and a location, and sometimes a different name or spelling. Bistritz for example is in Romania and is now written Bistrița (note the ț - it's Bistrița, not Bistrita), while Buda-Pesth is now written Budapest. But other cities like Munich are written in the same way in the 19th century as they are written today.
- Some kind of `NPC` type to represent the people in the book. This type could have a name, and also a way to track the places that the person visited.

To make a type inside a schema, just use the keyword `type` followed by the type name. Our `NPC` type will start out like this:

```sdl
type NPC;
```

That's all you need to create a type!

Now an `NPC` type that doesn't hold any properties isn't very useful, so we'll remove the `;` and replace them with `{}` curly brackets instead. The `NPC` type still doesn't have any properties but with the `{}` it is ready to hold them:

```sdl
type NPC {

}
```

But hold on, where is our schema and how do you put a schema together? The easiest way to create a schema is to start an EdgeDB project. Here's how to get started:

* First make sure you have [EdgeDB installed on your computer](https://www.edgedb.com/install). You will now have the EdgeDB CLI installed which allows you to make new projects, apply migrations, and do many other things.
* Once you have the EdgeDB CLI installed, make an empty directory where you want to hold your project. Now open up a command line (also known as a Terminal). If you are on Windows and haven't opened a command line before, type the Windows key and `cmd`, then click on Command Prompt. Then type `cd c:/easy-edgedb` or whatever the name of your directory is. Or you can right click on a folder inside Windows File Explorer and select `Open in Terminal`.
* Type `edgedb project init`. The EdgeDB CLI will now ask you a few questions. The output should look something like this:

```
No `edgedb.toml` found in `\\?\C:\easy-edgedb` or above
Do you want to initialize a new project? [Y/n]
> Y
Specify the name of EdgeDB instance to use with this project [default: easy_edgedb]:
> easy_edgedb
Checking EdgeDB versions...
Specify the version of EdgeDB to use with this project [default: 3.0]:
> 3.0
┌─────────────────────┬─────────────────────────────────────┐
│ Project directory   │ \\?\C:\easy-edgedb                  │
│ Project config      │ \\?\C:\easy-edgedb\edgedb.toml      │
│ Schema dir (empty)  │ \\?\C:\easy-edgedb\dbschema         │
│ Installation method │ WSL                                 │
│ Version             │ 3.0-rc.2+02561bd                    │
│ Instance name       │ easy_edgedb                         │
└─────────────────────┴─────────────────────────────────────┘
Version 3.0-rc.2+02561bd is already downloaded
Initializing EdgeDB instance...
Applying migrations...
Everything is up to date. Revision initial
Project initialized.
To connect to easy_edgedb, run `edgedb`
```

That was easy! Now try typing `edgedb`, which will take you into the EdgeDB REPL. (You can leave the REPL whenever you like by typing `\q`.) You can also type `edgedb ui` if you prefer to use the EdgeDB UI, which includes its own REPL and a lot of other nice tools like a data explorer and a query editor to help you put queries together. See [this short video](https://www.youtube.com/watch?v=iwnP_6tkKgc) for an introduction to the EdgeDB UI.

Now type `select 'Jonathan Harker';` and hit enter. You should see some output that looks like this, with your folder name instead of `db>` in the sample below:

```
db> select "Jonathan Harker";
{'Jonathan Harker'}
db>
```

You just made your first query in EdgeDB!

So now let's take a look at the schema so we can create our `NPC` type. The CLI has already let us know that the schema is inside the `\edgedb\dbschema` directory, so go inside there and look for a file called `default.esdl`. This is your schema. At the moment, all it holds is a module called `default`:

```sdl
module default {

}
```

A module is a space containing the types for our database, including our `NPC` type. Let's give it a try! Add the `NPC` type that we made above, as follows:

```sdl
module default {
  type NPC {
  }
}
```

Make sure the file is saved, and then create a migration. You can do this by typing `edgedb migration create` if you are on the command line, or (even easier) by typing `\migration create` inside the REPL. Commands that start with `edgedb` on the command line are done in the same way in the REPL except that you replace `edgedb` with a backslash.

You should see some output that looks like the following:

```
Created c:\easy-edgedb\dbschema\migrations\00001.edgeql, id: m14c35zu7lzg46r2337nwehaihkv6d4xwcatu6ulogd5kqafjtrnra
```

Now type `edgedb migrate` (or `\migrate` inside the REPL). The CLI will apply the migration with the following output:

```
Applied m14c35zu7lzg46r2337nwehaihkv6d4xwcatu6ulogd5kqafjtrnra (00001.edgeql)
```

And you are done!

To sum up, an EdgeDB migration simply consists of the following:

* Changing your schema,
* Typing `edgedb migration create` (or `\migration create`),
* Typing `edgedb migrate` (or `\migrate`).

We will be doing a lot of that in this book! Even by the second chapter you will be very used to this process.

```{eval-rst}
.. note::
  The CLI creates a new file upon each migration to generate the commands to change the schema to the one we want. The first file will be called 00001.edgeql, the second will be 00002.edgeql, and so on. These files are quite readable so feel free to take a look at if you are curious. But note that they use a syntax called DDL (Data Definition Language) that gives commands to EdgeDB one at a time, and you do not need to learn it. The language that we are using in the schema is called SDL (Schema Definition Language), which simply declares what a schema will look like. The CLI then automatically creates DDL commands to make it happen.
```

Now that we know how to do a schema migration, let's add some properties to our `NPC` type. Use the keyword `required` if the type needs it, and just the property name if it is optional. Let's give the `NPC` type a name and an array (a collection) of places visited:

```sdl
type NPC {
  required name: str;
  places_visited: array<str>;
}
```

With `required name` our `NPC` objects are always guaranteed to have a name - you can't make an `NPC` object without it. Give it a try with this query:

```edgedb
insert NPC;
```

You should see this error message:

```
error: MissingRequiredError: missing value for required property 'name' of object type 'default::NPC'
  ┌─ <query>:1:1
  │
1 │ insert NPC;
  │ ^^^^^^^^^^ error
```

The keyword `str` stands for string, which contains text. A `str` goes inside either single quotes: `select 'Jonathan Harker';` or double quotes: `select "Jonathan Harker";`.

If you have a single or double quote inside a string, you can use the `\` escape character to make EdgeDB treat it like just another letter inside the string. Give the `\` escape character a try to fix the query below that doesn't work because EdgeDB thinks that the single quote after `Harker` is the end of the string:

```edgeql
select 'Jonathan Harker's journal';
```

The next property is an `array`. An `array` is a collection of the same type, and our array here is an array of `str`s. We want it to look like this inside our first `NPC` object:

```
["Bistritz", "Munich", "Buda-Pesth"]
```

The idea is to easily search later and see which character has visited where.

Unlike `name`, `places_visited` is not a `required` property because we might later add minor characters that don't go anywhere. Maybe one person will be the "innkeeper_in_bistritz" or something, and we won't know or care about `places_visited` for him.

The next type to add to our schema is a `City`. It look like this:

```sdl
type City {
  required name: str;
  modern_name: str;
}
```

This type is pretty easy too, and just holds two properties with strings. The `modern_name` property is because the book Dracula was published in 1897 when spelling for cities was sometimes different. All cities have a name in the book (that's why it's `required`), and some will need a different modern name. But some don't: Munich in the book is still spelled Munich, for example. With this we could link the city names to their modern names so we can easily place them on a map.

## Migration

We've now added our first two types, but EdgeDB hasn't processed our changes yet. To make the changes happen we can do our first migration with `edgedb migration create`. After using this command, you will see a number of questions similar to the one below:

```
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
>
```

The CLI does this to make sure that it is applying the right changes to the schema, and to give us the opportunity to make changes if it has misunderstood anything. Most of the time the CLI guesses correctly and you will just end up clicking `y` (to mean yes) over and over again for each change. But there are other possible answers we can give. Type `?` if you want to see what the letters for the other possible answers mean.

One of those is `l`, which means:

```
l or list - list the DDL statements associated with prompt
```

Sure, let's be curious and try typing `l` to see what commands will be generated to create our `City` type. If we type `l` we will see the following:

```
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
> l
The following DDL statements will be applied:
  CREATE TYPE default::City {
      CREATE PROPERTY modern_name: std::str;
      CREATE REQUIRED PROPERTY name: std::str;
  };
```

Looks good! It's a type called `City` inside the module `default`, it has a required property called `name` and a property called `modern_name`.

So why didn't we need to write `property` inside the `City` type? Let's look at it again:

```sdl
type City {
  required name: str;
  modern_name: str;
}
```

There's no `property` keyword in there.

In the past you did have to write the `property` keyword for properties, and EdgeDB used a `->` (an arrow) instead of a `:` (a colon) after properties and links. Since the release of EdgeDB 3.0 in 2023 this is no longer required. So if you see any syntax that looks like the following, just keep in mind that it is pre-3.0 syntax:

```sdl
type City {
  # required name: str;            <-- Current syntax
  required property name -> str; # <-- Old syntax
  # modern name: str;              <-- Current syntax
  property modern_name -> str;   # <-- Old syntax
}
```

Now back to the schema. Let's now just type `y` for every question that the CLI gives us to see what it does. After typing `y` two times, the CLI asks us a sudden question:

```
Did you create object type 'default::City'? [y,n,l,c,b,s,q,?]
> y
Did you alter object type 'default::NPC'? [y,n,l,c,b,s,q,?]
> y
Please specify an expression to populate existing objects in order to make
property 'name' of object type 'default::NPC' required:
fill_expr>
```

The CLI is essentially saying: "There might be `NPC` objects in the database already. But now they all need to have a property called `name`, which wasn't required before. How should I decide what `name` to give them?"

Fortunately, the expression here is pretty simple: let's just give them all an empty string. Type `''` and hit enter, and the CLI will now be happy with the migration. Don't forget to complete the migration with `edgedb migration`, and we are done!

If you are curious, take a look inside the migration file that EdgeDB generated. You can see how it created a command to change the `NPC` type, including a default value for `name` for any objects in the database:

```
  ALTER TYPE default::NPC {
      CREATE REQUIRED PROPERTY name: std::str {
          SET REQUIRED USING (<std::str>{''});
      };
      CREATE PROPERTY places_visited: array<std::str>;
  };
```

There are a lot of other commands beyond the commands for migration, though we won't need them for this book. You could bookmark these four pages for later use, however:

- {ref}`Admin commands <docs:ref_cheatsheet_admin>`: Creating user roles, setting passwords, configuring ports, etc.
- {ref}`CLI commands <docs:ref_cheatsheet_cli>`: Creating databases, roles, setting passwords for roles, connecting to databases, etc.
- {ref}`REPL commands <docs:ref_cheatsheet_repl>`: Mostly shortcuts for a lot of the commands we'll be using in this book.
- {ref}`Various commands <docs:ref_eql_statements_rollback_tx>` about rolling back transactions, declaring savepoints, and so on.

There are also a few places to download packages to highlight your syntax if you like. EdgeDB has these packages available for [Atom](https://atom.io/packages/edgedb), [Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb), [Sublime Text](https://packagecontrol.io/packages/EdgeDB), and [Vim](https://github.com/edgedb/edgedb-vim).

And last but not least, here is the command to use if you want to destroy your project and start again.

```
edgedb instance destroy -I instance_name --force
```

Destroying an instance won't delete any files, so your schema and migration files won't be lost. If you want a completely clean slate, then first delete your instance, then delete the files inside the `migrations` folder, and then turn your schema back to `module default {}`.

Now let's start playing with some data!

## Selecting

The `select` keyword is the main query command in EdgeDB, and you use it to see results based on the input that comes after it. Keywords in EdgeDB are case insensitive, so `select`, `SELECT` and `SeLeCT` are all the same.

Let's give a `select` on a string a deeper look this time. Type `edgedb` to log into the REPL and give this a try:

```edgeql
select 'Jonathan Harker begins his journey.';
```

This returns `{'Jonathan Harker begins his journey.'}`, no surprise there. But did you notice that it is inside a `{}`? This `{}` is actually quite important: `{}` means that it's a set, and in fact {ref}`everything in EdgeDB is a set <docs:ref_eql_everything_is_a_set>` (make sure to remember that). Thanks to the "everything is a set" philosophy, EdgeDB doesn't have null. Where you would have null in other databases like SQL, EdgeDB just gives you an empty set: `{}`. The advantage here is that sets always behave the same way even when they are empty, instead of [all the unwelcome surprises](https://www.edgedb.com/blog/we-can-do-better-than-sql#null-a-bag-of-surprises) that come with using null.

For the next `select` queries, we will use some more operators that use the `=` sign:

- `:=` is used to declare / assign (to give a value to something),
- `=` is used to check equality (not `==`),
- `!=` is the opposite of `=`.

Let's use `:=` to assign a variable:

```edgeql
select jonathans_name := 'Jonathan Harker';
```

This just returns what we gave it: `{'Jonathan Harker'}`. But this time it's a string that we assigned called `jonathans_name` that is being returned.

Now let's do something with this variable. We can use the keyword `select` to use this variable and then compare it to `'Count Dracula'`:

```edgeql
select jonathans_name := 'Jonathan Harker' != 'Count Dracula';
```

You can read this as "Is the variable called jonathans_name, which is a string with the value 'Jonathan Harker', different from the string 'Count Dracula'?"

They are indeed different strings, so the output is `{true}`. Of course, you can just write `select 'Jonathan Harker' != 'Count Dracula';` for the same result. But knowing how to assign in this way with `:=` will allow us to do more complex operations later on.

## Inserting objects

Our database doesn't have any data in it yet! Time to change that.

Let's start inserting some `City` objects with the simple schema we already have by using the `insert` keyword, followed by the type name and some properties. The property `name` is required, while `modern_name` is not required.

Don't forget to separate each property by a comma, and finish the `insert` with a semicolon. Indentation isn't relevant in EdgeQL like it is in languages such as Python and F#, but EdgeDB prefers two spaces for indentation. Our first three cities look like this:

```edgeql
insert City { name := 'Munich' };

insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};
```

Note that a comma after the last parameter (a so-called "trailing comma") is optional - you can put it in or leave it out. In our three `City` inserts you can see that one insert has a trailing comma, while the others don't have it.

Finally, our `NPC` insert for Jonathan Harker would look like this, but don't insert it yet.

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Munich", "Buda-Pesth"],
};
```

Because hold on a second...that insert won't link our `NPC` object for Jonathan Harker to any of the `City` inserts that we already did! The `places_visited` parameter is just an array of strings. Is there a way to connect Jonathan Harker to our `City` objects instead?

Here's where our schema needs some improvement. To sum up:

- We have an `NPC` type and a `City` type,
- The `NPC` type has the property `places_visited` with the names of the cities, but they are just strings in an array. It would be better to link this property to the `City` type somehow.

So let's not do that `NPC` insert. We'll fix the `NPC` type soon by changing `array<str>` to a `multi City`, which will create a link to one or more `City` objects. After this change, `NPC` and `City` objects will actually be joined together.

But first let's look a bit closer at what happens when we use `insert`.

We saw when inserting our first three `City` objects that strings (`str`) are fine with unicode letters like ț. In fact, even emojis and special characters are just fine, so you could even create a `City` called `'🤠'` or `'(╯°□°)╯︵ ┻━┻'` if you wanted to. (Go ahead and insert a `City` called `'(╯°□°)╯︵ ┻━┻'` or anything else if you want - you won't break anything!)

EdgeDB has a somewhat similar type to `str` called a byte literal, which gives you the bytes of a string. This is mainly for raw data that humans don't need to view when saving to files. They must be characters that are 1 byte long.

You create byte literals by adding a `b` in front of the string:

```
db> select b'Bistritz';
{b'Bistritz'}
```

And because the characters must be 1 byte, only ASCII works for this type. So the name in `modern_name` as a byte literal will generate an error because of the `ț`:

```
db> select b'Bistrița';
error: EdgeQLSyntaxError: invalid bytes literal: character 'ț' is unexpected,
only ascii chars are allowed in bytes literals
  ┌─ <query>:1:8
  │
1 │ select b'Bistrița';
  │        ^ error
```

Now let's get to the output we saw from our three `City` object inserts. Did you notice that the database returned something every time we made an insert? In fact, EdgeDB gives you some information on the objects we work on every time we do an operation on them (such as `insert`), and the default output is the object's type with its `id` property, which is a `uuid`. UUID stands for [Universally Unique IDentifier](https://en.wikipedia.org/wiki/Universally_unique_identifier), and is used widely to make identifiers that won't be used anywhere else.

The output from each `City` insert looked like this:

```
{default::City {id: 462b29ea-ff3d-11eb-aeb7-b3cf3ba28fb9}}
```

This output is also what shows up when you use `select` to select a type. Just typing `select` with a type name will show you all the `uuid`s for the type that are in our database. Let's look at all the cities we have so far with the following `select` query:

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

So far so good, but this only tells us that there are three objects of type `City` and we can't see their names. To see inside them, we need to add property or link names to the query. This is called describing the {ref}`shape <docs:ref_eql_shapes>` of the data we want. For the next query we will select all `City` types again, but this time we will display their `modern_name`:

```edgeql
select City {
  modern_name,
};
```

Do you remember seeing that Munich doesn't have the property `modern_name`? But `modern_name` still shows up as an "empty set", because every value in EdgeDB is a set of elements, even if there's nothing inside. Here is the result:

```
{
  default::City {modern_name: {}},
  default::City {modern_name: 'Budapest'},
  default::City {modern_name: 'Bistrița'},
}
```

We know that the `City` object with `{}` for `modern_name` is Munich because we only have three objects so far, but to make the query clearer we should add `name` to the shape so that we can see that it's the city of Munich:

```edgeql
select City {
  name,
  modern_name
};
```

This gives the following output:

```
{
  default::City {name: 'Munich', modern_name: {}},
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

A `select` query doesn't have to return a full object though. If you just want to return a single property or link of an object, you can use `.` after the type name. For example, `select City.modern_name;` will return a set of strings for all of the `City` objects in the database so far. Note in the output that there is no `default::City` to be seen anywhere, because the query just returns a set of strings collected from the `modern_name` properties of every `City` object in the database.

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
  oh_and_by_the_way := 'A city in the book Dracula'
};
```

And here is the output:

```
{
  default::City {
    name_in_dracula: 'Munich',
    name_today: {},
    oh_and_by_the_way: 'A city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Buda-Pesth',
    name_today: 'Budapest',
    oh_and_by_the_way: 'A city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Bistritz',
    name_today: 'Bistrița',
    oh_and_by_the_way: 'A city in the book Dracula',
  },
}
```

If you need to give a parameter the same name as a keyword (like `insert`, `select`, and so on) you can enclose it in backticks. For example, if you wanted to call a parameter `select`, you would type \`select\`:

```edgeql
select City {
  name,
  `select` := "I like the word `select`"
};
```

The output is nice and clear, showing the word `select` without any backticks:

```
{
  default::City {name: 'Munich', select: 'I like the word `select`'},
  default::City {name: 'Buda-Pesth', select: 'I like the word `select`'},
  default::City {name: 'Bistritz', select: 'I like the word `select`'}
}
```

This brings up an interesting discussion about type safety. EdgeDB is strongly typed, meaning that everything needs a type and EdgeDB will not try to mix different types together. So this query will not work: 

```edgeql
db> select City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'A city in the book Dracula written in the year' + 1897
 };
```

EdgeDB refuses with the following message:

```
operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
```

But we didn't declare a type for `oh_and_by_the_way`, so how did EdgeDB know that it was a `str`?

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

In the next chapter we will learn how to change one type to another, but it's well past time for us to change our `NPC` type to link to `City`. Let's do that now.

## Links

It's finally time to change our property in `NPC` called `places_visited` to a link. Right now, `places_visited` gives us the names we want, but it makes more sense to link `NPC` and `City` together. After all, the `City` type has `.name` inside it which is a `str`. If we link `NPC` to `City` then we can get the `City` objects themselves but also access their names! We'll change `NPC` to look like this:

```sdl
type NPC {
  required name: str;
  multi places_visited: City;
}
```

We wrote `multi` in front of `places_visited` because one `NPC` should be able to link to more than one `City`. The opposite of `multi` is `single`, which only allows one object to link to it. But `single` is the default, so if you just write `places_visited: City` then EdgeDB will treat it as `single` link.

And now do a migration with `edgedb migration create` and `edgedb migrate`. The CLI questions this time are pretty easy:

```
c:\easy-edgedb>edgedb migration create
Connecting to an EdgeDB instance at localhost:10716...
Did you drop property 'places_visited' of object type 'default::NPC'?
[y,n,l,c,b,s,q,?]
> y
Did you create link 'places_visited' of object type 'default::NPC'?
[y,n,l,c,b,s,q,?]
> y
```

When we drop a property it also drops all the data for the property, so the CLI doesn't need to ask us for an expression. And adding a `multi` link just means that an `NPC` _can_ link to one or more `City` types, but they don't have to.

It's finally time to insert our `NPC` Jonathan Harker, and this time he can be connected to one or more `City` objects. Don't forget that `places_visited` is not `required`, so we could do an `insert` with just his name to create him. It would look like this:

```edgeql
insert NPC {
  name := 'Jonathan Harker',
};
```

If we were to do that, it would only create an `NPC` type connected to the `City` type but with nothing in it. And if we did a query on the `NPC` type as follows:

```edgeql
select NPC {
  name,
  places_visited
};
```

Then the output wouldn't show any links to any `City` objects:

```
{default::NPC {name: 'Jonathan Harker', places_visited: {}}}
```

But we want to have Jonathan be connected to the cities he has traveled to. So instead of just giving him a `name`, we'll also give him a `places_visited` multi link when we `insert` by writing `places_visited := City`:

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

Now let's see the places that Jonathan has visited. The query below is almost but not quite what we need:

```edgeql
select NPC {
  name,
  places_visited
};
```

Here is the output:

```
{
  default::NPC {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {id: 4ba1074e-ff3f-11eb-aeb7-cf15feb714ef},
      default::City {id: 4bab8188-ff3f-11eb-aeb7-f7b506bd047e},
      default::City {id: 4bacf860-ff3f-11eb-aeb7-97025b4d95af},
    },
  },
}
```

Close! But we didn't mention any properties inside `City` so we just got the object id numbers. Now we just need to let EdgeDB know that we want to see the `name` property of the `City` type as well. To do that, add a colon and then put `name` inside curly brackets.

```edgeql
select NPC {
  name,
  places_visited: {
    name
  }
};
```

Success! Now we get the output we wanted:

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
}
```

Interestingly, Jonathan Harker has been inserted with a link to every city in the database. In other words, `places_visited := City` means `places_visited := every City type in the database`. Right now we only have three `City` objects and one `NPC` object, and Jonathan Harker has visited them all. No problem so far. But later on we will have more cities and won't be able to just write `places_visited := City` for all the other characters. For that we will need the `filter` keyword, which we will learn to use in the next chapter.

Note that if you inserted "Jonathan Harker" multiple times (or any other `NPC` objects with the same name), you will now have multiple `NPC` objects with that name. The database doesn't give an error for this, because we haven't instructed it to keep an eye out for duplicates. In [Chapter 7](../chapter7/index.md) we will learn how to make sure the database doesn't allow multiple copies of `NPC` with the same name.

Remember how we saw that "each `.` in a path expression moves on to the next path, if there is another one to follow"? We can demonstrate that now that `NPC` links to `City`. The query below uses `.` two times. What do you think it will return?

```edgeql
select NPC.places_visited.modern_name;
```

Exactly, it will return a set of strings:

```
{'Budapest', 'Bistrița'}
```

In other words, it is a query that first follows the `places_visited` link inside `NPC`, which takes us to the three `City` objects, and then proceeds to their `modern_name` property which is a `str`. Two out of the three have a `modern_name` so the result is a set of two strings. You can use `.` as many times as you like as long as there is still a path to follow!

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
