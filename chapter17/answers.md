# Chapter 17 Questions and Answers

#### 1. How would you display every NPC's name, strength, population of cities visited, and age (displaying 0 if age = `{}`)? Try it on a single line.

This can be done on a single line but remember that we need an `[is City]` because `places_visited` links to `Place` and only `City` has population. It looks like this:

```edgeql
select (
  NPC.name,
  NPC.strength,
  (NPC.places_visited[is City].name, NPC.places_visited[is City].population),
  NPC.age if exists NPC.age else 0
);
```

#### 2. The query in 1. showed a lot of numbers without any context. What should we do?

The query for 1. gives us results like this:

```
{
  ('Mina Murray', 0, ('London', 3500000), 0),
  ('John Seward', 0, ('London', 3500000), 0),
  ('Quincey Morris', 0, ('London', 3500000), 0),
  # ...
}
```

The answer of course is named tuples:

```edgeql
select (
  name := NPC.name,
  strength := NPC.strength,
  city_populations := (NPC.places_visited[is City].name, NPC.places_visited[is City].population),
  age := NPC.age if exists NPC.age else 0
);
```

Now the output looks a lot better:

```
{
  (name := 'Mina Murray', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'John Seward', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'Quincey Morris', strength := 0, city_populations := ('London', 3500000), age := 0),
  # ...
}
```

You could even name the tuple inside the tuple with the city name and population if you wanted.

#### 3. How would you create an alias that contains all the seasons of the year?

Easy! Just make a set of scalars and precede it with the keyword `alias`. You could make it out of `str`s:

```sdl
alias Seasons := {
  'spring',
  'summer',
  'fall',
  'winter'
};
```

Or you could put together an enum instead:

```sdl
scalar type Season extending enum<Spring, Summer, Fall, Winter>;

alias Seasons := {
  Season.Spring,
  Season.Summer,
  Season.Fall,
  Season.Winter
};
```

#### 4. How do you make sure that no data is lost when changing a type's properties from owned properties to properties extended from abstract types?

EdgeDB will recognize that you intend to keep the same properties if the inherited properties have the same name and type. So if you have a type like this one that you would like to inherit its properties instead of owning them:

```sdl
type User {
  email: str;
  number: int64;
}
```

You can change the schema to look like this.

```sdl
type User extending HasEmail, HasNumber;

abstract type HasEmail {
  email: str;
}

abstract type HasNumber {
  number: int64;
}
```

You can take a look at the DDL in the created migration script to confirm this, which should look like this:

```
  ALTER TYPE default::User EXTENDING default::HasEmail,
  default::HasNumber LAST;
  ALTER TYPE default::User {
      ALTER PROPERTY email {
          DROP OWNED;
          RESET TYPE;
      };
      ALTER PROPERTY number {
          DROP OWNED;
          RESET TYPE;
      };
  };
```

The `ALTER PROPERTY` part shows that the property is just being altered, and not dropped.