# Chapter 11 Questions and Answers

#### 1. How would you write a function called `lucy()` that just returns all the `NPC` types matching the name 'Lucy Westenra'?

It's pretty simple, just don't forget to return `NPC` (or `Person`, depending on your preference):

```
function lucy() -> NPC
  using (
  SELECT NPC FILTER .name = 'Lucy Westenra');
```

Then you would use it as follows:

```
SELECT lucy() {
  name,
  places_visited: {name}
};
```

#### 2. How would you write a function that takes two strings and returns `Person` types with names that match it?

Let's call the function `get_two()`. It could look like this:

```
function get_two(one: str, two: str) -> SET OF Person
 using (
 with person_1 := (SELECT Person filter .name = one LIMIT 1),
 person_2 := (SELECT Person filter .name = two LIMIT 1),
 SELECT {person_1, person_2});
```

Here it is used for a regular query for John Seward, Count Dracula and their slaves:

```
SELECT get_two('John Seward', 'Count Dracula') {
  name,
  [IS Vampire].slaves: {name},
};
```

Here's the output:

```
{
  Object {name: 'John Seward', slaves: {}},
  Object {
    name: 'Count Dracula',
    slaves: {
      Object {name: 'Woman 1'},
      Object {name: 'Woman 2'},
      Object {name: 'Woman 3'},
    },
  },
}
```

#### 3. What will the output of this be?

Looking at the input, you can see that there will be 16 lines generated, because each part of each set will be combined with each part of every other set:

```
SELECT {'Jonathan', 'Arthur'} ++ {' loves '} ++ {'Mina', 'Lucy'} ++ {' but '} ++ {'Dracula', 'The inkeeper'} ++ {' doesn\'t love '} ++ {'Mina', 'Jonathan'};
```

(2 * 1 * 2 * 1 * 2 * 1 * 2 = 16)

That gives the following result:

```
{
  'Jonathan loves Mina but Dracula doesn\'t love Mina',
  'Jonathan loves Mina but Dracula doesn\'t love Jonathan',
  'Jonathan loves Mina but The inkeeper doesn\'t love Mina',
  'Jonathan loves Mina but The inkeeper doesn\'t love Jonathan',
  'Jonathan loves Lucy but Dracula doesn\'t love Mina',
  'Jonathan loves Lucy but Dracula doesn\'t love Jonathan',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Mina',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Mina but Dracula doesn\'t love Mina',
  'Arthur loves Mina but Dracula doesn\'t love Jonathan',
  'Arthur loves Mina but The inkeeper doesn\'t love Mina',
  'Arthur loves Mina but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Lucy but Dracula doesn\'t love Mina',
  'Arthur loves Lucy but Dracula doesn\'t love Jonathan',
  'Arthur loves Lucy but The inkeeper doesn\'t love Mina',
  'Arthur loves Lucy but The inkeeper doesn\'t love Jonathan',
}
```

#### 4. How would you make a function that tells you how many times larger one city is than another?

Here's one way to do it:

```
function two_cities(city_one: str, city_two: str) -> float64
  using (
    WITH first_city := (SELECT City filter .name = city_one),
         second_city := (SELECT City filter .name = city_two),
    SELECT first_city.population / second_city.population
);
```

Then it would be used in this sort of way:

```
edgedb> SELECT two_cities('Munich', 'Bistritz');
{25.277252747252746}
edgedb> SELECT two_cities('Munich', 'London');
{0.06572085714285714}
```

`city_one` and `city_two` could of course be a `City` type, but then it would take a lot more effort to use.
