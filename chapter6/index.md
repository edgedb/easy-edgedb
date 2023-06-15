---
tags: Filtering On Insert, Json
---

# Chapter 6 - Still no escape

> The women vampires are next to Jonathan and he can't move. Suddenly, Dracula runs into the room and tells the women to leave: "You can have him later, but not tonight!" The women listen to him, and leave.
>
> Jonathan wakes up in his bed the next day and it feels like a bad dream...but he sees that somebody folded his clothes, and he knows it was not just a dream. The castle has some visitors from Slovakia the next day, so Jonathan has an idea. He writes two letters, one to Mina and one to his boss. He gives the visitors some money and asks them to send the letters. But Dracula finds the letters and is angry. He burns the letters in front of Jonathan and tells him not to do that again. Jonathan is still stuck in the castle, and Dracula knows that Jonathan tried to trick him.

## Filtering on sets when doing an insert

There is not much new in this lesson when it comes to types, so let's look at improving our schema. Right now Jonathan Harker is still inserted like this:

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

This was fine when we only had cities, but now we have the `Place` and `Country` types. First we'll insert two more `Country` types to have some more variety:

```edgeql
insert Country {
  name := 'France'
};
insert Country {
  name := 'Slovakia'
};
```

(In Chapter 9 we'll learn how to do this with just one `insert`!)

Then we'll make a two new types called `Castle` for castles and castle towns, and `OtherPlace` for any other kind of place. They are super easy to make:

```sdl
type Castle extending Place;
type OtherPlace extending Place;
```

Put that into the schema and do a migration, and now we can insert our first `Castle`:

```edgeql
insert Castle {
  name := 'Castle Dracula'
};
```

We will insert some `OtherPlace` objects later on in the book. Now we have a good number of types from `Place` that aren't of the `City` type.

So back to Jonathan: in our database, he's been to four cities, one country, and one `Castle`...but he hasn't been to Slovakia or France, so we can't just insert him with `places_visited := select Place`. Instead, we can filter on `Place` against a set with the names of the places he has visited. If we were inserting Jonathan Harker for the first time, it would look like this:

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := (
    select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 
    'London', 'Romania', 'Castle Dracula'}
  )
};
```

```{eval-rst}
.. note::
  You'll notice that we just wrote the names in a set using `{}`, so we didn't need to use an array with `[]` to do it. (This is called a {ref}`set constructor <docs:ref_eql_set_constructor>`, by the way.)
```

But we already have a Jonathan Harker in the database. We could always do a quick `delete NPC filter .name = 'Jonathan Harker'` before doing this insert, but that's not ideal. Instead, we should do an update. For that we have the `update` and `set` keywords. The `update` keyword selects the type to start the update, and `set` is used to specify the parts that we want to change. So let's update Jonathan with this code instead:

```edgeql
update NPC 
filter .name = 'Jonathan Harker'
set {
  places_visited := (
    select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'}
  )
};
```

Now what if Jonathan ever escapes Castle Dracula and runs away to a new place? Let's pretend that the `PC` named Emil Sinclair changed the flow of the story by saving Jonathan and taking him away to Slovakia. In that case, we could select the `Place` object and use `+=` to add it to Jonathan's `places_visited` property:

```edgeql
update NPC
filter .name = 'Jonathan Harker'
set {
  places_visited += (select Place filter .name = 'Slovakia')
};
```

And do undo this change, you can just change `+=` to `-=` and run the command again. Let's do that, because Jonathan Harker doesn't ever actually end up visiting Slovakia.

EdgeDB will return the IDs of the objects that have been updated. In our case, it's just one:

```
{default::NPC {id: ca4e21c8-014f-11ec-9658-7f88bf45dae6}}
```

However, this doesn't mean that `places_visited` has been changed. If we had written something like `filter .name = 'SLLLovakia'` then the `set` portion of the update would have simply added an empty `{}` set to `places_visited`, because nothing matched the filter there. The returned `default::NPC` simply means that an `NPC` object was found that matched `filter .name = 'Jonathan Harker`, and that _something_ was done to it. But in this case, the _something_ was the addition of an empty set to an existing set, so nothing at all.

One quick way to ensure that `places_visited` is actually being updated with something is to use the `assert_exists` function. This function will pass on the output if it exists, but give an error if the output is an empty set. Here is the same `update` with the incorrect name 'SLLLovakia', which will now give an error:

```edgeql
update NPC
filter .name = 'Jonathan Harker'
set {
  places_visited += assert_exists((select Place filter .name = 'SLLovakia'))
};
```

Here is the output now:

```
edgedb error: CardinalityViolationError: assert_exists violation: expression returned an empty set
```

With that we now know {ref}`all three operators <docs:ref_eql_statements_update>` used after `set`: `:=`, `+=`, and `-=`.

Let's do another update. Remember the `lover` link on the `Person` type? Let's take a look at Jonathan and see how he is doing when it comes to love.

```edgeql
select Person {
  name,
  lover
} filter .name = 'Jonathan Harker';
```

Here's the output:

```
{default::NPC {name: 'Jonathan Harker', lover: {}}}
```

Ah, that's right. Mina Murray has Jonathan Harker as her `lover`, but Jonathan doesn't have her as his `lover` because we inserted him first before Mina Murray was inserted. We can change that now.

This command seems like it will work, but it doesn't quite. Do you remember why?

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  lover := assert_single(
    (select Person filter .name = 'Mina Murray')
  )
};
```

No error is generated, but let's look at Jonathan Harker after the update:

```edgeql
select Person { name, lover } filter .name = 'Jonathan Harker';
```

Surprisingly, the output is `{default::NPC {name: 'Jonathan Harker', lover: {}}}`!

After a bit of thought, we remember that we learned the `detached` keyword in Chapter 4 when inserting Mina. But let's imagine that we forgot how `detached` works and are trying to figure out what is going on. Let's investigate exactly what happens when `detached` doesn't get used.

First we'll do a quick `select` to see what's going on. We'll select Jonathan Harker and also add a computed `lover := (select Person filter .name = 'Mina Murray')` inside the shape to see what shows up:

```edgeql
select Person {
 name,
 lover := (select Person filter .name = 'Mina Murray')
 } filter .name = 'Jonathan Harker';
```

Again, the output is `{default::NPC {name: 'Jonathan Harker', lover: {}}}`. That in itself is a hint that something didn't work properly. Let's try another `select`. This time we will simply make `lover` into a `(select Person {name})` to see everything that shows up before the filter. The query now looks like this:

```edgeql
select Person {
 name,
 lover := (select Person {name})
 } filter .name = 'Jonathan Harker';
```

This time, the output is _very_ interesting: `{default::NPC {name: 'Jonathan Harker', lover: default::NPC {name: 'Jonathan Harker'}}}`. This output proves that the `select Person` inside this query effectively means to select the `Person` object or objects that have already been selected.

So now let's do the update properly with the `detached` keyword so that Jonathan can finally be connected to Mina. (After all, he has enough to worry about without needing to think about this too.)

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  lover := assert_single(
    (select detached Person filter .name = 'Mina Murray')
  )
};
```

Let's do a `select` query now to make sure that it worked:

```edgeql
select Person {name, lover: {name}} filter .name = 'Jonathan Harker';
```

The output is now `{default::NPC {name: 'Jonathan Harker', lover: default::NPC {name: 'Mina Murray'}}}`. Success!

Now, if you use `update` without `filter` it will do the same change on all the types. This update below for example would give every `Person` type every single `Place` in the database under `places_visited`:

```edgeql
update Person
set {
  places_visited := Place
};
```

## Concatenation with ++

One other operator is `++`, which does concatenation (joining together) instead of adding.

You can do simple operations with it like: `select 'My name is ' ++ 'Jonathan Harker';` which gives `{'My name is Jonathan Harker'}`. Or you can do more complicated concatenations as long as you continue to join strings to strings:

```edgeql
select 'A character from the book: ' ++ (select NPC.name) 
       ++ ', who is not ' ++ (select Vampire.name);
```

This prints:

```
{
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
  'A character from the book: Jonathan Harker, who is not Count Dracula',
}
```

The concatenation operator works on arrays too, putting them into a single array. So the output of:

```edgeql
select ['I', 'am'] ++ ['Jonathan', 'Harker'];
```

Will be:

```
{['I', 'am', 'Jonathan', 'Harker']}
```

The last type that the concatenation operator works on is bytes.

## Using `insert` instead of `select` when adding links

Let's also change the `Vampire` type to link it to `MinorVampire` from that side instead. You'll remember that Count Dracula is the only real vampire, while the others are of type `MinorVampire`. In the book, any person that Dracula bites becomes his slave - once they die, they become another vampire that lives forever that Dracula can control with his mind (that's what makes the book scary). That means we need a `multi` link:

```sdl
type Vampire extending Person {
  age: int16;
  multi slaves: MinorVampire;
}
```

Then we can `insert` the `MinorVampire` type at the same time as we insert the information for Count Dracula. But first let's remove `link master` from `MinorVampire`, because we don't want two objects linking to each other in this way. There are two reasons for that:

- When we declare a `Vampire` it has `slaves`, but if there are no `MinorVampire`s yet then it will be empty: {}. And if we declare the `MinorVampire` type first it has a `master`, but if we declare them first then their `master` (a `required` link) will not be there.
- If both types link to each other, we won't be able to delete them if we need to. The error looks something like this:

```edgeql-repl
db> delete MinorVampire;
  ERROR: ConstraintViolationError: deletion of default::MinorVampire
  (ee6ca100-006f-11ec-93a9-4b5d85e60114) is prohibited by link target policy
  Detail: Object is still referenced in link slave of default::Vampire
  (e5ef5bc6-006f-11ec-93a9-77e907d251d6).
db> delete Vampire;
ERROR: ConstraintViolationError: deletion of default::Vampire
  (e5ef5bc6-006f-11ec-93a9-77e907d251d6) is prohibited by link target policy
  Detail: Object is still referenced in link master of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114).
```

So first we simply change `MinorVampire` to a type extending `Person`:

```sdl
type MinorVampire extending Person {
}
```

And then do a migration.

We said we would leave Dracula alone after all the practice deleting him before...but let's delete him again. Also let's delete 'Vampire Woman 1' (the `MinorVampire`) so we can practise inserting them all at the same time. Here's what the insert looks like:

```edgeql
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (insert MinorVampire {
      name := 'Vampire Woman 1',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 2',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 3',
    }),
  }
};
```

Note two things: we used `{}` after `slaves` to put everything into a set, and each `insert` is inside `()` brackets to capture the insert.

Now we don't have to insert the `MinorVampire` types first and then filter: we can just put them in together with Dracula.

Then when we `select Vampire` like this:

```edgeql
select Vampire {
  name,
  slaves: {name}
};
```

We have a nice output that shows them all together:

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Vampire Woman 1'},
      default::MinorVampire {name: 'Vampire Woman 2'},
      default::MinorVampire {name: 'Vampire Woman 3'},
    },
  },
}
```

This might make you wonder: what if we do want two-way links? There's actually a very cool way to do it called a **backlink** that will let us give `MinorVampire` a link based on what links _to_ it, but we won't look at it until Chapters 14 and 15. If you're really curious you can skip to those chapters but there's a lot more to learn before then.

## Just type \<json> to generate json

What do we do if we want the same output in JSON? It couldn't be easier: just cast using `<json>`. Any type in EdgeDB can be cast to JSON this easily:

```edgeql
select <json>Vampire {
  # <json> is the only difference from the select above
  name,
  slaves: {name}
};
```

This will transform the results into JSON. However, what the REPL will show by default looks more like this:

```
{
  Json("{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Vampire Woman 1\"}, {\"name\": \"Vampire Woman 2\"}, {\"name\": \"Vampire Woman 3\"}]}"),
}
```

Let's go through the result together. The outer curly braces are just telling you that what's inside is one or more results returned by the query. Then the actual result is a string containing JSON. Because the JSON part is inside a string all the `"` there need to be escaped, so they appear as `\"`.

To make REPL show JSON in a nicer format just type `\set output-format json-pretty`. Then the results will look more familiar:

```json
{
  "name": "Count Dracula",
  "slaves": [{"name": "Vampire Woman 1"}, {"name": "Vampire Woman 2"}, {"name": "Vampire Woman 3"}]
}
```

Now to get back to the default format, we can type `\set output-format default`.
To keep things easy to read, this book will show JSON output using this `json-pretty` output format.

## Converting back from JSON

So what about the other way around, namely JSON to an EdgeDB type? You can do this too, but remember to think about the JSON type that you are giving to cast. The EdgeDB philosophy is that casts should be symmetrical: a type cast into JSON should only be cast back into that type. For example, here is the first date in the book Dracula as a string, then cast to JSON and then into a `cal::local_date`:

```edgeql
select <cal::local_date><json>'18930503';
```

This is fine because `<json>` turns it into a JSON string, and `cal::local_date` can be created from a string. The result we get is `{<cal::local_date>'1893-05-03'}`. But if we try to turn the JSON value into an `int64`, it won't work:

```edgeql
select <int64><json>'18930503';
```

The problem is that it is a conversion from a JSON string to an EdgeDB `int64`. It gives this error: `edgedb error: InvalidValueError: expected JSON number or null; got JSON string`. To keep things symmetrical, you need to cast a JSON string to an EdgeDB `str` and then cast into an `int64`:

```edgeql
select <int64><str><json>'18930503';
```

Now it works: we get `{18930503}` which began as an EdgeDB `str`, turned into a JSON string, then back into an EdgeDB `str`, and finally was cast into an `int64`.

The {ref}`documentation on JSON <docs:ref_std_json>` explains which JSON types turn into which EdgeDB types, lists functions for working with JSON values, and is good to bookmark if you need to convert from JSON a lot.

[Here is all our code so far up to Chapter 6.](code.md)

<!-- quiz-start -->

## Time to practice

1. This select is incomplete. How would you complete it so that it says "Pleased to meet you. I'm " and then the NPC's name followed by a period?

   ```edgeql
   select NPC {
     name,
     greeting := ## Put the rest here
   };
   ```

2. How would you update Mina's `places_visited` to include Romania if she went to Castle Dracula for a visit?

3. With the set `{'W', 'J', 'C'}`, how would you display all the `Person` types with a name that contains any of these capital letters?

   Hint: it involves `with` and a bit of concatenation.

4. How would you display this same query as JSON?

5. How would you add ' the Great' to every Person type?

   Bonus question: what's a quick way to undo this using string indexing?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan climbs the castle wall to get into the Count's room._
