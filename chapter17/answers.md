# Chapter 17 Questions and Answers

#### 1. How would you display every NPC's name, strength, population of cities visited, and age (displaying 0 if age = `{}`)? Try it on a single line.

This can be done on a single line but remember that we need an `[IS City]` because `places_visited` links to `Place` and only `City` has population. It looks like this:

```edgeql
SELECT (
  NPC.name,
  NPC.strength,
  (NPC.places_visited[IS City].name, NPC.places_visited[IS City].population),
  NPC.age if EXISTS NPC.age else 0
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
SELECT (
  name := NPC.name,
  strength := NPC.strength,
  city_populations := (NPC.places_visited[IS City].name, NPC.places_visited[IS City].population),
  age := NPC.age if EXISTS NPC.age else 0
);
```

Now the output looks a lot better:

```
{
  (name := 'Mina Murray', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'John Seward', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'Quincey Morris', strength := 0, city_populations := ('London', 3500000), age := 0),
}
```

You could even name the tuple inside the tuple with the city name and population if you wanted.

#### 3. Renfield is now dead and needs a `last_appearance`. Try writing a function called `make_dead(person_name: str, date: str) -> Person` that lets you just write the character name and date to do it.

Here's how you could do it:

```sdl
function make_dead(person_name: str, date: str) ->  Person {
  using (
    WITH dead_person := (
        SELECT default::Person
        FILTER (.name = person_name)
        LIMIT 1
      )
    UPDATE dead_person
    SET {
      last_appearance := <cal::local_date>date
    }
  );
```
