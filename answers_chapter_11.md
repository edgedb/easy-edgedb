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
