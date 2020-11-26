# Chapter 4 Questions and Answers

#### 1. This insert is not working.

The keyword to make it work is `DETACHED`, so that we can pull from the `NPC` type in general instead of the `NPC` type we are inserting:

```
INSERT NPC {
  name := 'I Love Mina',
  lover := (SELECT NPC FILTER .name LIKE '%Mina%' LIMIT 1)
};
```

One other method that can work is this:

```
INSERT NPC {
  name := 'I Love Mina',
  lover := (SELECT Person FILTER .name LIKE '%Mina%' LIMIT 1)
};
```

We changed `SELECT NPC` to `SELECT Person`. Even though `NPC` extends from `Person`, `Person` is a different type so we can draw from it without needing to write `DETACHED`.

#### 2. How would you display up to 2 `Person` types (and their `name` property) whose names include the letter `a`?

Like this, using `LIKE` and `LIMIT`:

```
SELECT Person {
  name
} FILTER .name LIKE '%a%' LIMIT 2;
```

#### 3. How would you display all the `Person` types (and their names) that have never visited anywhere?

Just use `NOT EXISTS`:

```
SELECT NPC {name} FILTER NOT EXISTS .places_visited;
```

#### 4. How to display `{true}` if cal::local_time has a 9 and `{false}` otherwise?

Take the original query:

```
SELECT has_nine_in_it := <cal::local_time>'09:09:09';
```

and use <str> to cast the `cal::local_time` to a string, then add `LIKE '%9%' at the end, like so:
  
```
SELECT has_nine_in_it := <str><cal::local_time>'09:09:09' LIKE '%9%';
```

#### 5. Selecting while inserting and adding a property called `age_ten_years_later`

First take the original insert:
```
INSERT NPC {
  name := "The Innkeeper's Son",
  age := 10
};
```

then wrap it in parentheses, add a SELECT and remove the `;`

```
SELECT(INSERT NPC {
  name := "The Innkeeper's Son",
  age := 10
});
```

Now just add the fields like in any other `SELECT`, then add `age_ten_years_later`:

```
SELECT(INSERT NPC {
    name := "The Innkeeper's Son",
    age := 10
 }) {
  name,
  age,
  age_ten_years_later := .age + 10
};
```

This gives us: `{Object {name: 'The Innkeeper\'s Son', age: 10, age_ten_years_later: 20}}`
