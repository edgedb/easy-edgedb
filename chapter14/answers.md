# Chapter 14 Questions and Answers

#### 1. How would you display just the numbers for all the `Person` types? e.g. if there are 20 of them, displaying `1, 2, 3..., 18, 19, 20`.

One way to do it is with `enumerate()`. This is easy if we are just starting from 0, since `enumerate()` gives a tuple with two items. The first one is an `int64` so we select that:

```
SELECT enumerate(Person).0;
```

That will display: `{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19}`

Warning: selecting `.1` will generate an error since that is where the rest of the object type is, and there is no way to properly display that. But displaying a single property is fine, such as in this example:

```
SELECT enumerate(Person.strength).1;
```

So now how to display starting from 1? With a `FOR` loop:

```
WITH numbers := enumerate(Person).0,
 FOR number in {numbers}
 UNION(
 SELECT number + 1);
```

Don't forget to write `number in {numbers}` so that `numbers` becomes a set.

#### 2. Using a reverse lookup, how would you display 1) all the `Place` types (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

This is not too hard if you start it in steps, first with a filter to get all the `Place` types:

```
SELECT Place {
  name
  } FILTER .name LIKE '%o%';
```

And here they are:

```
{
  default::Country {name: 'Romania'},
  default::Country {name: 'Slovakia'},
  default::City {name: 'London'},
}
```

Now we'll add the reverse lookup to the same query, and call the computable `visitors`:

```
SELECT Place {
 name,
 visitors := .<places_visited[IS Person].name
 } FILTER .name LIKE '%o%';
```

Now we can see who visited:

```
{
  default::Country {name: 'Romania', visitors: {}},
  default::Country {name: 'Slovakia', visitors: {}},
  default::City {
    name: 'London',
    visitors: {
      'Lucy Westenra',
      'The innkeeper',
      'Mina Murray',
      'John Seward',
      'Quincey Morris',
      'Arthur Holmwood',
      'Renfield',
      'Abraham Van Helsing',
      'Emil Sinclair',
    },
  },
}
```

A clear victory for London as the most visited place! If you wanted, you could also add a `visitor_numbers := .<places_visited[IS Person].name` to it to get the number of visitors too.

#### 3. Using reverse lookup, how would you display all the Person types that will later become `MinorVampire`s?

We can do it with a computable again, which we'll call `later_vampire`. Then we use reverse lookup to link back to the `MinorVampire` that links to `Person` via the property `former_self`:

```
SELECT Person {
  name,
  later_vampire := .<former_self[IS MinorVampire].name
  } FILTER exists .later_vampire;
```

That just gives us Lucy:

`{default::NPC {name: 'Lucy Westenra', later_vampire: {'Lucy Westenra'}}}`

This is not bad, but we can probably do better - `later_vampire` here isn't telling us anything about the type. Let's add some type info:

```
SELECT Person {
   name,
   later_vampire := .<former_self[IS MinorVampire] {
     name,
     __type__: {
       name
       }
     }
   } FILTER exists .later_vampire;
```

Now we can see that `later_vampire` is of type `MinorVampire` instead of just displaying a string:

```
{
  default::NPC {
    name: 'Lucy Westenra',
    later_vampire: {
      default::MinorVampire {
        name: 'Lucy Westenra',
        __type__: schema::ObjectType {name: 'default::MinorVampire'},
      },
    },
  },
}
```

#### 4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

First you would create the annotation itself, since it's not available yet:

```
abstract annotation note;
```

After that you would just put it into the `MinorVampire` type and it's done!

```
type MinorVampire extending Person {
    link former_self -> Person;
    annotation note := 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type';
}
```

#### 5. How would you see this `note` annotation for `MinorVampire` in a query?

It would look like this:

```
SELECT (INTROSPECT MinorVampire) {
  name,
  annotations: {
    name,
    @value
  }
};
```

Here's the output:

```
{
  schema::ObjectType {
    name: 'default::MinorVampire',
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type',
      },
    },
  },
}
```

Of course, `@value` by itself will show you the strings for all the annotations, but we just have one here.
