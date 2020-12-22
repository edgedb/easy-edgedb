# Chapter 14 - Jonathan Harker returns

> Finally there is some good news: Jonathan Harker is alive. After escaping Castle Dracula, he found his way to Budapest in August and then to a hospital, which sent Mina a letter. The hospital tells Mina that "He has had some fearful shock and continues to talk about wolves and poison and blood, of ghosts and demons." Mina takes a train to the hospital where Jonathan was recovering, and they take a train back to England to the city of Exeter where they get married. Mina sends Lucy a letter from Exeter about the good news...but it arrives too late and Lucy never opens it. Meanwhile, the men visit the graveyard as planned and see vampire Lucy walking around. When Arthur sees her he finally believes Van Helsing, and so do the rest. They now know that vampires are real, and manage to destroy her. Arthur is sad but happy to see that Lucy is no longer forced to be a vampire and can now die in peace.

So we have a new city called Exeter, and adding it is of course easy:

```
INSERT City {
  name := 'Exeter',
  population := 40000
};
```

That's the population of Exeter at the time, and it doesn't have a `modern_name` that is different from the one in the book.

## Adding annotations to types and using @

Now that we know how to do introspection queries, we can start to give our types `annotations`. An annotation is a string inside the type definition that gives us information about it. By default, annotations can use the titles `title` or `description`.

Let's imagine that in our game a `City` needs at least 50 buildings. Let's use `description` for this:

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

Ah, of course: the `annotations: {name}` part returns the name of the _type_, which is `std::description`. This is where `@` comes in. To get the value inside we write something else: `@value`. The `@` is used to directly access the value inside (the string) instead of just the type name. Let's try one more time:

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

What if we want an annotation with a different name besides `title` and `description`? That's easy, just declare with `abstract annotation` inside the schema and give it a name. We want to add a warning so that's what we'll call it:

```
abstract annotation warning;
```

We'll imagine that it is important to use `Castle` instead of `OtherPlace` for not just castles, but castle towns too. Thanks to the new abstract annotation, now `OtherPlace` gives that information along with the other annotation:

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

## Even more working with dates

A lot of characters are starting to die now, so let's think about that. We could come up with a method to see who is alive and who is dead, depending on a `cal::local_date`. First let's take a look at the `People` objects we have so far. We can easily count them with `SELECT count(Person)`, which gives `{23}`.

There is also a function called [`enumerate()`](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::enumerate) that gives tuples of the index and the set that we give it. We'll use this to compare to our `count()` function to make sure that our number is right.

First a simple example of how to use `enumerate()`:

```
WITH three_things := {'first', 'second', 'third'},
 SELECT enumerate(three_things);
```

The output is:

```
{(0, 'first'), (1, 'second'), (2, 'third')}
```

So now let's use it with `SELECT enumerate(Person.name);` to make sure that we have 23 results. The last index should be 22:

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

There are only 18? Oh, that's right: the `Crewman` objects don't have a name so they don't show up. How can we get them in the query? We could of course try something fancy like this:

```
WITH
  a := array_agg((SELECT enumerate(Person.name))),
  b:= array_agg((SELECT enumerate(Crewman.number))),
  SELECT (a, b);
```

(`array_agg()` is to avoid multiplying sets by sets, as we saw in Chapter 12)

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

The `Crewman` types are now just numbers, which doesn't look good. Let's give up on fancy queries and just update them with names based on the numbers instead. This will be easy:

```
UPDATE Crewman
  SET {
    name := 'Crewman ' ++ <str>.number
};
```

So now that everyone has a name, let's use that to see if they are dead or not. The logic is simple: we input a `cal::local_date`, and if it's greater than the date for `last_appearance` then the character is dead.

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

## Reverse links

Finally, let's look at how to follow links in reverse direction, one of EdgeDB's most powerful and useful features. Learning to use it can take a bit of effort, but it's well worth it.

We know how to get Count Dracula's `slaves` by name with something like this:

```
SELECT Vampire {
 name,
 slaves: {
   name
   }
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

But what if we are doing the opposite? Namely, starting from `SELECT MinorVampire` and wanting to access the `Vampire` type connected to it. Because right now, we can only bring up the properties that belong to the `MinorVampire` and `Person` type. Consider the following:

```
SELECT MinorVampire {
  name,
  # master... how do we get this?
  # There's no link to Vampire inside MinorVampire...
}
```

Since there's no `link master -> Vampire`, how do we go backwards to see the `Vampire` type that links to it?

This is where reverse links come in, where we use `.<` instead of `.` and specify the type we are looking for: `[IS Vampire]`.

First let's move out of our `MinorVampire` query and just look at how `.<` works. Here is one example:

```
SELECT MinorVampire.<slaves[IS Vampire] {
  name,
  age
  };
```

Because it goes in reverse order, it is selecting `Vampire` that has `.slaves` that are of type `MinorVampire`.

You can think of `MinorVampire.<slaves[IS Vampire] {name, age}` as "Select the name and age of the Vampire type with slaves that are of type MinorVampire" - from right to left.

Here is the output:

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```

So far that's the same as just `SELECT Vampire: {name, age}`. But it becomes very useful in our query before, where we wanted to access multiple types. Now we can select all the `MinorVampire` types and their master:

```
SELECT MinorVampire {
  name,
  master := .<slaves[IS Vampire] {name},
};
```

You could read `.<slaves[IS Vampire] {name}` as "the name of the `Vampire` type that links back to `MinorVampire` through `.slaves`".

Here is the output:

```
{
  default::MinorVampire {
    name: 'Lucy Westenra',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 1',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 2',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 3',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
}
```

[Here is all our code so far up to Chapter 14.](code.md)

## Time to practice

<!-- quiz-start -->

1. How would you display just the numbers for all the `Person` types? e.g. if there are 20 of them, displaying `1, 2, 3..., 18, 19, 20`.

2. Using reverse lookup, how would you display 1) all the `Place` types (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

3. Using reverse lookup, how would you display all the Person types that will later become `MinorVampire`s?

   Hint: Remember, `MinorVampire` has a link back to the vampire's former self.

4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

5. How would you see this `note` annotation for `MinorVampire` in a query?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 15: [Time to get revenge.](../chapter15/index.md)
