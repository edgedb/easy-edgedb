# Chapter 9 Questions and Answers

#### 1. Why doesn't this insert work and how can it be fixed?

It needs to be a set instead of an array, so change the brackets to `{}` instead:

```edgeql
for castle in {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
union (
  insert Castle {
    name := castle
  }
);
```

#### 2. How would you do the same insert while displaying the castle's name at the same time?

It looks like this:

```edgeql
select (
  for castle in {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
  union (
    insert Castle {
      name := castle
    }
  )
) { name };
```

#### 3. How would you change the `Vampire` type if all vampires needed a minimum strength of 10?

Since `strength` comes from `abstract type Person`, you would need to overload it and give it a constraint. The `Vampire` type would then look like this:

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire;
  overloaded strength: int16 {
    constraint min_value(10)
  }
}
```

#### 4. How would you update all the `Person` types to show that they died on September 11, 1893?

You could give them each a `last_appearance` using the `to_local_date` function that we used a lot in this chapter:

```edgeql
update Person
set {
  last_appearance := cal::to_local_date(1893, 9, 11),
};
```

Or you could just cast to a `cal::local_date` from a string:

```edgeql
update Person
set {
  last_appearance := <cal::local_date>'1893-09-11'
};
```

#### 5. All the `Person` characters that have an `e` or an `a` in their name have been brought back to life. How would you update to do this?

You can just `update` by using `like` on a set instead of a single letter:

```edgeql
update Person filter .name like {'%a%', '%e%'}
set {
  last_appearance := {}
};
```

And if you wanted to display the results at the same time to make sure, it would look like this:

```edgeql
select (
  update Person filter .name like {'%a%', '%e%'}
  set {
    last_appearance := {}
  }
) {
  name,
  last_appearance
};
```
