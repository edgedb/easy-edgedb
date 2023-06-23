# Chapter 11 Questions and Answers

#### 1. How would you write a function called `get_lucies()` that just returns all the `Person` types matching the name 'Lucy Westenra'?

It's pretty simple, just don't forget to return a `set of Person` because there might be more than one:

```sdl
function get_lucies() -> set of Person
  using (
    select Person filter .name = 'Lucy Westenra'
  );
```

Then you would use it as follows:

```edgeql
select get_lucies() {
  name,
  places_visited: {name}
};
```

There is an exclusive constraint on the `name` property for NPC, so only one NPC can have the name 'Lucy Westenra'. However, the constraint is delegated from the abstract type Person, meaning that each concrete type that extends Person will have that constraint separately. This means that you can add 'Lucy Westenra' the `Sailor` and then this function will return both the Sailor and the NPC named 'Lucy Westenra'.

#### 2. How would you write a function that takes two strings and returns `Person` types with names that match it?

Let's call the function `get_two()`. It could look like this:

```sdl
function get_two(one: str, two: str) -> set of Person
  using (
    with person_1 := (select Person filter .name = one),
         person_2 := (select Person filter .name = two),
    select {person_1, person_2}
  );
```

Here it is used for a regular query for John Seward, Count Dracula and their slaves:

```edgeql
select get_two('John Seward', 'Count Dracula') {
  name,
  [is Vampire].slaves: {name},
};
```

Here's the output:

```
{
  default::NPC {name: 'John Seward', slaves: {}},
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

#### 3. What will the output of this be?

Looking at the input, you can see that there will be 16 lines generated, because each part of each set will be combined with each part of every other set:

```edgeql
select 
  {'Jonathan', 'Arthur'} ++ 
  {' loves '} ++ 
  {'Mina', 'Lucy'} ++ 
  {' but '} ++ 
  {'Dracula', 'The inkeeper'} ++ 
  {' doesn\'t love '} ++ 
  {'Mina', 'Jonathan'};
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

```sdl
function two_cities(city_one: str, city_two: str) -> float64
  using (
    with first_city := (select City filter .name = city_one),
         second_city := (select City filter .name = city_two),
    select first_city.population / second_city.population
  );
```

Then it would be used in this sort of way:

```
db> select two_cities('Munich', 'Bistritz');
{25.277252747252746}
db> select two_cities('Munich', 'London');
{0.06572085714285714}
```

`city_one` and `city_two` could of course be a `City` type, but then it would take a lot more effort to use.

#### 5. Will `select (City.population + City.population)` and `select ((select City.population) + (select City.population))` produce different results?

Yes. The first one is a `select` on a single set that adds each item to itself, giving this: `{28800, 460046, 805412, 18200, 7000000}`

Meanwhile, the second is two sets where each item is added to each other one in every possible combination. That gives 25 results in total:

```
{
  28800,
  244423,
  417106,
  23500,
  3514400,
  244423,
  460046,
  632729,
  239123,
  3730023,
  417106,
  632729,
  805412,
  411806,
  3902706,
  23500,
  239123,
  411806,
  18200,
  3509100,
  3514400,
  3730023,
  3902706,
  3509100,
  7000000,
}
```
