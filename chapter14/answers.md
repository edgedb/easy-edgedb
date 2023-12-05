# Chapter 14 Questions and Answers

#### 1. How would you create a global str that tells you whether vampires are currently asleep or awake?

We already have a `global time` that returns a single `Time` object, so this can be used to make a global called something like `vampire_state` that is a computable referencing the `vampires_are` property inside `time`, which is an enum that we created before:

```sdl
scalar type SleepState extending enum <Asleep, Awake>;
```

You could make it this way:

```
global vampire_state := (select 'Vampires are now ' ++ <str>(global time.vampires_are));
```

Or for bonus points, add the `str_lower` function to change `Asleep` or `Awake` output to lowercase:

```
global vampire_state := (select 'Vampires are now ' ++ str_lower(<str>(global time.vampires_are)));
```

#### 2. Using a computed backlink, how would you display 1) all the `Place` types (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

This is not too hard if you start it in steps, first with a filter to get all the `Place` types:

```edgeql
select Place {
  name
} filter .name like '%o%';
```

And here they are:

```edgeql
{
  default::Country {name: 'Romania'},
  default::Country {name: 'Slovakia'},
  default::City {name: 'London'},
}
```

Now we'll add the computed backlink to the same query, and call the link `visitors`:

```edgeql
select Place {
  name,
  visitors := .<places_visited[is Person].name
} filter .name like '%o%';
```

Now we can see who visited:

```
{
  default::Country {name: 'Romania', visitors: {'Jonathan Harker', 'Count Dracula'}},
  default::Country {name: 'Slovakia', visitors: {}},
  default::City {
    name: 'London',
    visitors: {
      'Jonathan Harker',
      'The innkeeper',
      'Mina Murray',
      'Lucy Westenra',
      'John Seward',
      'Quincey Morris',
      'Arthur Holmwood',
      'Renfield',
      'Abraham Van Helsing',
    },
  },
}
```

A clear victory for London as the most visited place! If you wanted, you could also add a `visitor_numbers := count(.<places_visited[is Person].name)` to it to get the number of visitors too.

#### 3. Using a computed backlink, how would you display all the Person types that will later become `MinorVampire`s?

We can do it with a computed link again, which we'll call `later_vampire`. Then we use a computed backlink to link back to the `MinorVampire` that links to `Person` via the link `former_self`:

```edgeql
select Person {
  name,
  later_vampire := .<former_self[is MinorVampire].name
} filter exists .later_vampire;
```

That just gives us Lucy:

`{default::NPC {name: 'Lucy Westenra', later_vampire: {'Lucy'}}}`

This is not bad, but we can probably do better - `later_vampire` here isn't telling us anything about the type. Let's add some type info:

```edgeql
select Person {
  name,
  later_vampire := .<former_self[is MinorVampire] {
    name,
    __type__: {
      name
    }
  }
} filter exists .later_vampire;
```

Now we can see that `later_vampire` is of type `MinorVampire` instead of just displaying a string:

```
{
  default::NPC {
    name: 'Lucy Westenra',
    later_vampire: {
      default::MinorVampire {
        name: 'Lucy',
        __type__: schema::ObjectType {name: 'default::MinorVampire'},
      },
    },
  },
}
```

#### 4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

First you would create the annotation itself, since it's not available yet:

```sdl
abstract annotation note;
```

After that you would just put it into the `MinorVampire` type and it's done!

```sdl
type MinorVampire extending Person {
  former_self: Person;
  annotation note := 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type';
}
```

#### 5. How would you see this `note` annotation for `MinorVampire` in a query?

It would look like this:

```edgeql
select (introspect MinorVampire) {
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
        name: 'default::note',
        @value: 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type',
      },
    },
  },
}
```

Of course, `@value` by itself will show you the strings for all the annotations, but we just have one here.
