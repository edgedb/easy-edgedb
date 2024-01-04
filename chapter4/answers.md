# Chapter 4 Questions and Answers

#### 1. This insert is not working.

The keyword to make it work is `detached`, so that we can pull from the `NPC` type in general instead of the `NPC` type we are inserting:

```edgeql
insert NPC {
  name := 'I Love Mina',
  lover := assert_single(
    (select detached NPC filter .name like '%Mina%')
  )
};
```

One other method that can work is this:

```edgeql
insert NPC {
  name := 'I Love Mina',
  lover := assert_single(
    (select Person filter .name like '%Mina%')
  )
};
```

We changed `select NPC` to `select Person`. Even though `NPC` extends from `Person`, `Person` is a different type so we can draw from it without needing to write `detached`.

#### 2. How would you display up to 2 `Person` types (and their `name` property) whose names include the letter `a`?

Like this, using `like` and `limit`:

```edgeql
select Person {
  name
} filter .name like '%a%' limit 2;
```

#### 3. How would you display all the `Person` types (and their names) that have never visited anywhere?

Just use `not exists`:

```edgeql
select Person {name} filter not exists .places_visited;
```

#### 4. How to display `{true}` if cal::local_time has a 9 and `{false}` otherwise?

Take the original query:

```edgeql
select has_nine_in_it := <cal::local_time>'09:09:09';
```

and use `<str>` to cast the `cal::local_time` to a string, then add `like '%9%' at the end, like so:

```edgeql
select has_nine_in_it := <str><cal::local_time>'09:09:09' like '%9%';
```

#### 5. Selecting while inserting and adding a property called `age_ten_years_later`

First take the original insert:

```edgeql
insert NPC {
  name := "The Innkeeper's Son",
  age := 10
};
```

then wrap it in parentheses, add a select and remove the `;`

```edgeql
select (
  insert NPC {
    name := "The Innkeeper's Son",
    age := 10
  }
);
```

Now just add the fields like in any other `select`, then add `age_ten_years_later`:

```edgeql
select (
  insert NPC {
    name := "The Innkeeper's Son",
    age := 10
  }
) {
  name,
  age,
  age_ten_years_later := .age + 10
};
```

This gives us: `{default::NPC {name: 'The Innkeeper\'s Son', age: 10, age_ten_years_later: 20}}`
