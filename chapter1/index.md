---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# Chapter 1 - Jonathan Harker travels to Transylvania

In the beginning of the book we see the main character Jonathan Harker, a young lawyer who is going to meet a client. The client is a rich man named Count Dracula who lives somewhere in Eastern Europe. Jonathan doesn't yet know that Count Dracula is a vampire, so he's enjoying the trip to a new part of Europe. The book begins with Jonathan writing in his journal as he travels. The parts that are good for a database are in **bold**:

> **3 May**. **Bistritz**.—Left **Munich** at **8:35 P.M.**, on **1st May**, arriving at **Vienna** early next morning; should have arrived at 6:46, but train was an hour late. **Buda-Pesth** seems a wonderful place, from the glimpse which I got of it from the train...

## Schema, object types

This is already a lot of information, and it helps us start to think about our database schema. The language used for EdgeDB is called EdgeQL, and is used to define, mutate, and query data. Inside it is {ref}`SDL (schema definition language)<docs:ref_eql_sdl>` that makes migration easy, and which we will learn in this book. So far our schema needs the following:

- Some kind of City or Location type. These types that we can create are called {ref}`object types <docs:ref_datamodel_object_types>`, made out of properties and links. What properties should a City type have? Perhaps a name and a location, and sometimes a different name or spelling. Bistritz for example is now called Bistrița (it's in Romania), and Buda-Pesth is now written Budapest.
- Some kind of Person type. We need it to have a name, and also a way to track the places that the person visited.

To make a type inside a schema, just use the keyword `type` followed by the type name, then `{}` curly brackets. Our `Person` type will start out like this:

```sdl
type Person {
}
```

That's all you need to create a type, but there's nothing inside there yet. Inside the brackets we add the properties for our `Person` type. Use `required property` if the type needs it, and just `property` if it is optional.

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

An `array` is a collection of the same type, and our array here is an array of `str`s. We want it to look like this: `["Bistritz", "Vienna", "Buda-Pesth"]`. The idea is to easily search later and see which character has visited where.

`places_visited` is not a `required` property because we might later add minor characters that don't go anywhere. Maybe one person will be the "innkeeper_in_bistritz" or something, and we won't know or care about `places_visited` for him.

Now for our City type:

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
}
```

This is similar, just properties with strings. The book Dracula was published in 1897 when spelling for cities was sometimes different. All cities have a name in the book (that's why it's `required`), but some won't need a different modern name. Vienna is still Vienna, for example. We are imagining that our game will link the city names to their modern names so we can easily place them on a map.

## Migration

We haven't created our database yet, though. There are two small steps that we need to do first [after installing EdgeDB](https://www.edgedb.com/download). First we create a {ref}`"project" <docs:ref_quickstart_createdb>` that makes it easier to keep track of the schema and deal with migrations. Then we just open a console to our database by running `edgedb`, which will connect us to the default database called "edgedb". We'll use that a lot for experimenting.

Sometimes it's useful to create a whole new database to try something out. You can do that with the `CREATE DATABASE` keyword and our name for it:

```edgeql
CREATE DATABASE dracula;
```

Then we type `\c dracula` to connect to it. And you can type `\c edgedb` to get back to the default one.

Lastly, we need to do a migration. This will give the database the structure we need to start interacting with it. Migrations are not difficult with EdgeDB's {ref}`built-in tools <docs:ref_cli_edgedb_migration>`. However, we will use a {ref}`console shortcut <docs:ref_eql_ddl_migrations>` instead:

- First you start them with `START MIGRATION TO {}`
- Inside this you add at least one `module`, so your types can be accessed. A module is a namespace, a place where similar types go together. The part on the left side of the `::` is the name of the module, and the type inside is to the right. If you wrote `module default` and then `type Person`, the type `Person` would be at `default::Person`. So when you see a type like `std::bytes` for example, this means the type `bytes` inside `std` (the standard library).
- Then you add the types we mentioned above, and finish up the block by ending with a `}`. Then outside of that, type `POPULATE MIGRATION` to add the data.
- Finally, you type `COMMIT MIGRATION` and the migration is done.

There are naturally a lot of other commands beyond this, though we won't need them for this book. You could bookmark these four pages for later use, however:

- {ref}`Admin commands <docs:ref_cheatsheet_admin>`: Creating user roles, setting passwords, configuring ports, etc.
- {ref}`CLI commands <docs:ref_cheatsheet_cli>`: Creating databases, roles, setting passwords for roles, connecting to databases, etc.
- {ref}`REPL commands <docs:ref_cheatsheet_repl>`: Mostly shortcuts for a lot of the commands we'll be using in this book.
- {ref}`Various commands <docs:ref_eql_statements_rollback_tx>` about rolling back transactions, declaring savepoints, and so on.

There are also a few places to download packages to highlight your syntax if you like. EdgeDB has these packages available for [Atom](https://atom.io/packages/edgedb), [Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb), [Sublime Text](https://packagecontrol.io/packages/EdgeDB), and [Vim](https://github.com/edgedb/edgedb-vim).

So here's the `City` type we just made:

```edgeql
type City {
  required property name -> str;
  property modern_name -> str;
}
```

## Selecting

Here are three operators in EdgeDB that have the `=` sign:

- `:=` is used to declare,
- `=` is used to check equality (not `==`),
- `!=` is the opposite of `=`.

Let's try them out with `SELECT`. `SELECT` is the main query command in EdgeDB, and you use it to see results based on the input that comes after it.

By the way, keywords in EdgeDB are case insensitive, so `SELECT`, `select` and `SeLeCT` are all the same. But using capital letters is the normal practice for databases so we'll continue to use them that way.

First we'll just select a string:

```edgeql
SELECT 'Jonathan Harker\'s journey begins.';
```

This returns `{'Jonathan Harker\'s journey begins.'}`, no surprise there. Did you notice that it's returned inside a `{}`? The `{}` means that it's a set, and in fact {ref}`everything in EdgeDB is a set <docs:ref_eql_everything_is_a_set>` (make sure to remember that). It's also why EdgeDB doesn't have null: where you would have null in other languages, EdgeDB just gives you an empty set: `{}`.

Next we'll use `:=` to assign a variable:

```edgeql
SELECT jonathans_name := 'Jonathan Harker';
```

This just returns what we gave it: `{'Jonathan Harker'}`. But this time it's a string that we assigned called `jonathans_name` that is being returned.

Now let's do something with this variable. We can use the keyword `WITH` to use this variable and then compare it to `'Count Dracula'`:

```edgeql
WITH jonathans_name := 'Jonathan Harker',
SELECT jonathans_name != 'Count Dracula';
```

The output is `{true}`. Of course, you can just write `SELECT 'Jonathan Harker' != 'Count Dracula'` for the same result. Soon we will actually do something with the variables we assign with `:=`.

## Inserting objects

Let's get back to the schema. Later on we can think about adding time zones and locations for the cities for our imaginary game. But in the meantime, we will add some items to the database using `INSERT`.

Don't forget to separate each property by a comma, and finish the `INSERT` with a semicolon. EdgeDB also prefers two spaces for indentation.

```edgeql
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

Note that a comma at the end is optional - you can put it in or leave it out. Here we put a comma at the end sometimes and left it out at other times to show this.

Finally, the `Person` insert would look like this:

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Vienna", "Buda-Pesth"],
};
```

But hold on a second. That insert won't link it to any of the `City` inserts that we already did. Here's where our schema needs some improvement:

- We have a `Person` type and a `City` type,
- The `Person` type has the property `places_visited` with the names of the cities, but they are just strings in an array. It would be better to link this property to the `City` type somehow.

So let's not do that `Person` insert. We'll fix the `Person` type soon by changing `array<str>` from a `property` to something called `multi link` to the `City` type. This will actually join them together.

But first let's look a bit closer at what happens when we use `INSERT`.

As you can see, `str`s are fine with unicode letters like ț. Even emojis and special characters are just fine: you could even create a `City` called '🤠' or '(╯°□°)╯︵ ┻━┻' if you wanted to.

EdgeDB also has a byte literal type that gives you the bytes of a string. This is mainly for raw data that humans don't need to view such when saving to files. They must be characters that are 1 byte long.

You create byte literals by adding a `b` in front of the string:

```edgeql-repl
edgedb> SELECT b'Bistritz';
{b'Bistritz'}
```

And because the characters must be 1 byte, only ASCII works for this type. So the name in `modern_name` as a byte literal will generate an error because of the `ț`:

```edgeql-repl
edgedb> SELECT b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
  ┌─ query:1:8
  │
1 │ SELECT b'Bistrița';
  │        ^ error
```

Every time you `INSERT` an item, EdgeDB gives you a `uuid` back. That's the unique number for each item. It will look like this:

```
{default::Person {id: 462b29ea-ff3d-11eb-aeb7-b3cf3ba28fb9}}
```

It is also what shows up when you use `SELECT` to select a type. Just typing `SELECT` with a type will show you all the `uuid`s for the type. Let's look at all the cities we have so far:

```edgeql
SELECT City;
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
SELECT City {
  modern_name,
};
```

Once again, you don't need the comma after `modern_name` because it's at the end of the query.

You will remember that one of our cities (Vienna) doesn't have anything for `modern_name`. But it still shows up as an "empty set", because every value in EdgeDB is a set of elements, even if there's nothing inside. Here is the result:

```
{default::City {modern_name: {}}, default::City {modern_name: 'Budapest'}, default::City {modern_name: 'Bistrița'}}
```

So there is some object with an empty set for `modern_name`, while the other two have a name. This shows us that EdgeDB doesn't have `null` like in some languages: if nothing is there, it will return an empty set.

The first object is a mystery so we'll add `name` to the query so we can see that it's the city of Vienna:

```edgeql
SELECT City {
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

If you just want to return a single part of a type without the object structure, you can use `.` after the type name. For example, `SELECT City.modern_name` will give this output:

```
{'Budapest', 'Bistrița'}
```

This type of expression is called a _path expression_ or a _path_, because it is the direct path to the values inside. And each `.` moves on to the next path, if there is another one to follow.

You can also change property names like `modern_name` to any other name if you want by using `:=` after the name you want. Those names you choose become the variable names that are displayed. For example:

```edgeql
SELECT City {
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
SELECT City {
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

Also note that `oh_and_by_the_way` is of type `str` even though we didn't have to tell it. EdgeDB is strongly typed: everything needs a type and it will not try to mix them together. So if you write `SELECT 'Jonathan Harker' + 8;` it will simply refuse with an error: `operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
`.

On the other hand, it can use "type inference" to guess the type, and that is what it does here: it knows that we are creating a `str`. We will look at changing types and working with different types soon.

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
INSERT Person {
  name := 'Jonathan Harker',
};
```

But this would only create a `Person` type connected to the `City` type but with nothing in it. Let's see what's inside:

```edgeql
SELECT Person {
  name,
  places_visited
};
```

Here is the output: `{default::Person {name: 'Jonathan Harker', places_visited: {}}}`

But we want to have Jonathan be connected to the cities he has traveled to. We'll change `places_visited` when we `INSERT` to `places_visited := City`:

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

We haven't filtered anything, so it will put all the `City` types in there. Now let's see the places that Jonathan has visited. The code below is almost but not quite what we need:

```edgeql
SELECT Person {
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
SELECT Person {
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

Of course, Jonathan Harker has been inserted with a connection to every city in the database. Right now we only have three `City` objects, so this is no problem yet. But later on we will have more cities and won't be able to just write `places_visited := City` for all the other characters. For that we will need `FILTER`, which we will learn to use in the next chapter.

[Here is all our code so far up to Chapter 1.](code.md)

<!-- quiz-start -->

## Time to practice

1. Entering the code below returns an error. Try adding one character to make it return `{true}`.

   ```edgeql
   WITH my_name = 'Timothy',
   SELECT my_name != 'Benjamin';
   ```

2. Try inserting a `City` called Constantinople, but now known as İstanbul.
3. Try displaying all the names of the cities in the database. (Hint: you can do it in a single line of code and won't need `{}` to do it)
4. Try selecting all the `City` types along with their `name` and `modern_name` properties, but change `.name` to say `old_name` and change `modern_name` to say `name_now`.
5. Will typing `SelecT City;` produce an error?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan Harker arrives in Romania._
