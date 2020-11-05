Chapter 11 Questions and Answers

1. How would you write a function called `lucy()` that just returns all the `NPC` types matching the name 'Lucy Westenra'?

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
