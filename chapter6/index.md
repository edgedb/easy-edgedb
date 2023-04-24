---
tags: Filtering On Insert, Json
---

# Chapter 6 - Still no escape

> The women vampires are next to Jonathan and he can't move. Suddenly, Dracula runs into the room and tells the women to leave: "You can have him later, but not tonight!" The women listen to him. Jonathan wakes up in his bed and it feels like a bad dream...but he sees that somebody folded his clothes, and he knows it was not just a dream. The castle has some visitors from Slovakia the next day, so Jonathan has an idea. He writes two letters, one to Mina and one to his boss. He gives the visitors some money and asks them to send the letters. But Dracula finds the letters, and is angry. He burns them in front of Jonathan and tells him not to do that again. Jonathan is still stuck in the castle, and Dracula knows that Jonathan tried to trick him.

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

Then we'll make a new type called `OtherPlace` for places that aren't cities or countries. That's easy: `type OtherPlace extending Place;` and it's done.

Then we'll insert our first `OtherPlace`:

```edgeql
insert OtherPlace {
  name := 'Castle Dracula'
};
```

That gives us a good number of types from `Place` that aren't of the `City` type.

So back to Jonathan: in our database, he's been to four cities, one country, and one `OtherPlace`...but he hasn't been to Slovakia or France, so we can't just insert him with `places_visited := select Place`. Instead, we can filter on `Place` against a set with the names of the places he has visited. It looks like this:

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := (
    select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'}
  )
};
```

You'll notice that we just wrote the names in a set using `{}`, so we didn't need to use an array with `[]` to do it. (This is called a {ref}`set constructor <docs:ref_eql_set_constructor>`, by the way.)

Now what if Jonathan ever escapes Castle Dracula and runs away to a new place? Let's pretend that he escapes and runs away to Slovakia. Of course, we can change his `insert` signature to include `'Slovakia'` in the set of names. But what do we do to make a quick update? For that we have the `update` and `set` keywords. The `update` keyword selects the type to start the update, and `set` is for the parts we want to change. It looks like this:

```edgeql
update NPC
filter .name = 'Jonathan Harker'
set {
  places_visited += (select Place filter .name = 'Slovakia')
};
```

You'll know that it succeeded because EdgeDB will return the IDs of the objects that have been updated. In our case, it's just one:

```
{default::NPC {id: ca4e21c8-014f-11ec-9658-7f88bf45dae6}}
```

And if we had written something like `filter .name = 'SLLLovakia'` then it would return `{}`, letting us know that nothing matched. Or to be precise: the top-level object matched on `filter .name = 'Jonathan Harker'`, but `places_visited` doesn't get updated because nothing matched the `filter` there.

And since Jonathan hasn't visited Slovakia, we can use `-=` instead of `+=` with the same `update` syntax to remove it now.

With that we now know {ref}`all three operators <docs:ref_eql_statements_update>` used after `set`: `:=`, `+=`, and `-=`.

Let's do another update. Remember this?

```edgeql
select Person {
  name,
  lover
} filter .name = 'Jonathan Harker';
```

Mina Murray has Jonathan Harker as her `lover`, but Jonathan doesn't have her because we inserted him first. We can change that now:

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  lover := assert_single(
    (select detached Person filter .name = 'Mina Murray')
  )
};
```

We need to use `detached Person` here for the same reason we needed it when using `insert NPC` to create Mina Murray with her lover. We're using `update` on one `Person` and we also need to `select` another `Person` as lover. Now `link lover` for Jonathan finally shows Mina instead of an empty `{}`.

Of course, if you use `update` without `filter` it will do the same change on all the types. This update below for example would give every `Person` type every single `Place` in the database under `places_visited`:

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
select 'A character from the book: ' ++ (select NPC.name) ++ ', who is not ' ++ (select Vampire.name);
```

This prints:

```
{
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
  'A character from the book: Jonathan Harker, who is not Count Dracula',
}
```

(The concatenation operator works on arrays too, putting them into a single array. So `select ['I', 'am'] ++ ['Jonathan', 'Harker'];` gives `{['I', 'am', 'Jonathan', 'Harker']}`.)

Let's also change the `Vampire` type to link it to `MinorVampire` from that side instead. You'll remember that Count Dracula is the only real vampire, while the others are of type `MinorVampire`. That means we need a `multi link`:

```sdl
type Vampire extending Person {
  property age -> int16;
  multi link slaves -> MinorVampire;
}
```

Then we can `insert` the `MinorVampire` type at the same time as we insert the information for Count Dracula. But first let's remove `link master` from `MinorVampire`, because we don't want two objects linking to each other. There are two reasons for that:

- When we declare a `Vampire` it has `slaves`, but if there are no `MinorVampire`s yet then it will be empty: {}. And if we declare the `MinorVampire` type first it has a `master`, but if we declare them first then their `master` (a `required link`) will not be there.
- If both types link to each other, we won't be able to delete them if we need to. The error looks something like this:

```edgeql-repl
edgedb> DELETE MinorVampire;
ERROR: ConstraintViolationError: deletion of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114) is prohibited by link target policy
  Detail: Object is still referenced in link slave of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6).
edgedb> DELETE Vampire;
ERROR: ConstraintViolationError: deletion of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6) is prohibited by link target policy
  Detail: Object is still referenced in link master of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114).
```

So first we simply change `MinorVampire` to a type extending `Person`:

```sdl
type MinorVampire extending Person {
}
```

and then we create them all together with Count Dracula like this:

```edgeql
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (insert MinorVampire {
      name := 'Woman 1',
    }),
    (insert MinorVampire {
      name := 'Woman 2',
    }),
    (insert MinorVampire {
      name := 'Woman 3',
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
      default::MinorVampire {name: 'Woman 1'},
      default::MinorVampire {name: 'Woman 2'},
      default::MinorVampire {name: 'Woman 3'},
    },
  },
}
```

This might make you wonder: what if we do want two-way links? There's actually a very convenient way to do it (it's called a **backward link**), but we won't look at it until Chapters 14 and 15. If you're really curious you can skip to those chapters but there's a lot more to learn before then.

## Just type \<json> to generate json

What do we do if we want the same output in json? It couldn't be easier: just cast using `<json>`. Any type in EdgeDB (except `bytes`) can be cast to json this easily:

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
  "{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Woman 1\"}, {\"name\": \"Woman 2\"}, {\"name\": \"Woman 3\"}]}",
}
```

Let's go through the result together. The outer curly braces are just telling you that what's inside is one or more results returned by the query. Then the actual result is a string containing JSON. Because the JSON part is inside a string all the `"` there need to be escaped, so they appear as `\"`.

To make REPL show JSON in a nicer format just type `\set output-format json-pretty`. Then the results will look more familiar:

```json
{
  "name": "Count Dracula",
  "slaves": [{"name": "Woman 1"}, {"name": "Woman 2"}, {"name": "Woman 3"}]
}
```

To restore the default format type: `\set output-format default`.


## Converting back from JSON

So what about the other way around, namely JSON to an EdgeDB type? You can do this too, but remember to think about the JSON type that you are giving to cast. The EdgeDB philosophy is that casts should be symmetrical: a type cast into JSON should only be cast back into that type. For example, here is the first date in the book Dracula as a string, then cast to JSON and then into a `cal::local_date`:

```edgeql
select <cal::local_date><json>'18870503';
```

This is fine because `<json>` turns it into a JSON string, and `cal::local_date` can be created from a string. The result we get is `{<cal::local_date>'1887-05-03'}`. But if we try to turn the JSON value into an `int64`, it won't work:

```edgeql
select <int64><json>'18870503';
```

The problem is that it is a conversion from a JSON string to an EdgeDB `int64`. It gives this error: `ERROR: InvalidValueError: expected json number or null; got json string`. To keep things symmetrical, you need to cast a JSON string to an EdgeDB `str` and then cast into an `int64`:

```edgeql
select <int64><str><json>'18870503';
```

Now it works: we get `{18870503}` which began as an EdgeDB `str`, turned into a JSON string, then back into an EdgeDB `str`, and finally was cast into an `int64`.

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
