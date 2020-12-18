# Chapter 6 - Still no escape

> The women vampires are next to Jonathan and he can't move. Suddenly, Dracula runs into the room and tells the women to leave: "You can have him later, but not tonight!" The women listen to him. Jonathan wakes up in his bed and it feels like a bad dream...but he sees that somebody folded his clothes, and he knows it was not just a dream. The castle has some visitors from Slovakia the next day, so Jonathan has an idea. He writes two letters, one to Mina and one to his boss. He gives the visitors some money and asks them to send the letters. But Dracula finds the letters, and is angry. He burns them in front of Jonathan and tells him not to do that again. Jonathan is still stuck in the castle, and Dracula knows that Jonathan tried to trick him.

## Filtering on sets when doing an insert

There is not much new in this lesson when it comes to types, so let's look at improving our schema. Right now Jonathan Harker is still inserted like this:

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

This was fine when we only had cities, but now we have the `Place` and `Country` types. First we'll insert two more `Country` types to have some more variety:

```edgeql
INSERT Country {
  name := 'France'
};
INSERT Country {
  name := 'Slovakia'
};
```

(In Chapter 9 we'll learn how to do this with just one `INSERT`!)

Then we'll make a new type called `OtherPlace` for places that aren't cities or countries. That's easy: `type OtherPlace extending Place;` and it's done.

Then we'll insert our first `OtherPlace`:

```edgeql
INSERT OtherPlace {
  name := 'Castle Dracula'
};
```

That gives us a good number of types from `Place` that aren't of the `City` type.

So back to Jonathan: in our database, he's been to four cities, one country, and one `OtherPlace`...but he hasn't been to Slovakia or France, so we can't just insert him with `places_visited := SELECT Place`. Instead, we can filter on `Place` against a set with the names of the places he has visited. It looks like this:

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};
```

You'll notice that we just wrote the names in a set using `{}`, so we didn't need to use an array with `[]` to do it. (This is called a [set constructor](https://www.edgedb.com/docs/edgeql/expressions/overview/#set-constructor), by the way.)

Now what if Jonathan ever escapes Castle Dracula and runs away to a new place? Let's pretend that he escapes and runs away to Slovakia. Of course, we can change his `INSERT` signature to include `'Slovakia'` in the set of names. But what do we do to make a quick update? For that we have the `UPDATE` and `SET` keywords. `UPDATE` selects the type to start the update, and `SET` is for the parts we want to change. It looks like this:

```edgeql
UPDATE NPC
FILTER .name = 'Jonathan Harker'
SET {
  places_visited += (SELECT Place FILTER .name = 'Slovakia')
};
```

You'll know that it succeeded because EdgeDB will return the IDs of the objects that have been updated. In our case, it's just one:

```
{
  Object { id: <uuid>"6f436006-3f65-11eb-b6de-a3e7cc8efd4f" }
}
```

And if we had written something like `FILTER .name = 'SLLLovakia'` then it would return `{}`, letting us know that nothing matched.

And since Jonathan hasn't visited Slovakia, we can use `-=` instead of `+=` with the same `UPDATE` syntax to remove it now.

With that we now know [all three operators](https://www.edgedb.com/docs/edgeql/statements/update) used after `SET`: `:=`, `+=`, and `-=`.

Let's do another update. Remember this?

```edgeql
SELECT Person {
  name,
  lover
} FILTER .name = 'Jonathan Harker';
```

Mina Murray has Jonathan Harker as her `lover`, but Jonathan doesn't have her because we inserted him first. We can change that now:

```edgeql
UPDATE Person FILTER .name = 'Jonathan Harker'
SET {
  lover := (SELECT Person FILTER .name = 'Mina Murray' LIMIT 1)
};
```

Now `link lover` for Jonathan finally shows Mina instead of an empty `{}`.

Of course, if you use `UPDATE` without `FILTER` it will do the same change on all the types. This update below for example would give every `Person` type every single `Place` in the database under `places_visited`:

```edgeql
UPDATE Person
SET {
  places_visited := Place
};
```

## Concatenation with ++

One other operator is `++`, which does concatenation (joining together) instead of adding.

You can do simple operations with it like: `SELECT 'My name is ' ++ 'Jonathan Harker';` which gives `{'My name is Jonathan Harker'}`. Or you can do more complicated concatenations as long as you continue to join strings to strings:

```edgeql
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

```sdl
type Vampire extending Person {
  property age -> int16;
  multi link slaves -> MinorVampire;
}
```

Then we can `INSERT` the `MinorVampire` type at the same time as we insert the information for Count Dracula. But first let's remove `link master` from `MinorVampire`, because we don't want two objects linking to each other. There are two reasons for that:

- When we declare a `Vampire` it has `slaves`, but if there are no `MinorVampire`s yet then it will be empty: {}. And if we declare the `MinorVampire` type first it has a `master`, but if we declare them first then their `master` (a `required link`) will not be there.
- If both types link to each other, we won't be able to delete them if we need to. The error looks something like this:

```
ERROR: ConstraintViolationError: deletion of default::Vampire (cc5ee436-fa23-11ea-85e0-e78b548f5a59) is prohibited by link target policy

DETAILS: Object is still referenced in link master of default::MinorVampire (cc87c78e-fa23-11ea-85e0-8f5149329e3a).
```

So first we simply change `MinorVampire` to a type extending `Person`:

```sdl
type MinorVampire extending Person {
}
```

and then we create them all together with Count Dracula like this:

```edgeql
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

Note two things: we used `{}` after `slaves` to put everything into a set, and each `INSERT` is inside `()` brackets to capture the insert.

Now we don't have to insert the `MinorVampire` types first and then filter: we can just put them in together with Dracula.

Then when we `select Vampire` like this:

```edgeql
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

This might make you wonder: what if we do want two-way links? There's actually a very convenient way to do it (it's called a **backward link**), but we won't look at it until Chapters 14 and 15. If you're really curious you can skip to those chapters but there's a lot more to learn before then.

## Just type \<json> to generate json

What do we do if we want the same output in json? It couldn't be easier: just cast using `<json>`. Any type in EdgeDB (except `bytes`) can be cast to json this easily:

```edgeql
SELECT <json>Vampire {
      # <json> is the only difference from the SELECT above
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

## Converting back from JSON

So what about the other way around, namely JSON to an EdgeDB type? You can do this too, but remember to think about the JSON type that you are giving to cast. The EdgeDB philosophy is that casts should be symmetrical: a type cast into JSON should only be cast back into that type. For example, here is the first date in the book Dracula as a string, then cast to JSON and then into a `cal::local_date`:

```edgeql
SELECT <cal::local_date><json>'18870503';
```

This is fine because `<json>` turns it into a JSON string, and `cal::local_date` can be created from a string. The result we get is `{<cal::local_date>'1887-05-03'}`. But if we try to turn the JSON value into an `int64`, it won't work:

```edgeql
SELECT <int64><json>'18870503';
```

The problem is that it is a conversion from a JSON string to an EdgeDB `int64`. It gives this error: `ERROR: InvalidValueError: expected json number, null; got json string`. To keep things symmetrical, you need to cast a JSON string to an EdgeDB `str` and then cast into an `int64`:

```edgeql
SELECT <int64><str><json>'18870503';
```

Now it works: we get `{18870503}` which began as an EdgeDB `str`, turned into a JSON string, then back into an EdgeDB `str`, and finally was cast into an `int64`.

The [documentation on JSON](https://www.edgedb.com/docs/datamodel/scalars/json) explains which JSON types turn into which EdgeDB types and is good to bookmark if you need to convert from JSON a lot.

[Here is all our code so far up to Chapter 6.](code.md)

## Time to practice

<!-- quiz-start -->

1. This select is incomplete. How would you complete it so that it says "Pleased to meet you, I'm " and then the NPC's name?

   ```edgeql
   SELECT NPC {
     name,
     greeting := ## Put the rest here
   };
   ```

2. How would you update Mina's `places_visited` to include Romania if she went to Castle Dracula for a visit?

3. With the set `{'W', 'J', 'C'}`, how would you display all the `Person` types with a name that contains any of these capital letters?

   Hint: it involves `WITH` and a bit of concatenation.

4. How would you display this same query as JSON?

5. How would you add ' the Great' to every Person type?

   Bonus question: what's a quick way to undo this using string indexing?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 7: [Jonathan climbs the castle wall to try to get into Count Dracula's room.](../chapter7/index.md)
