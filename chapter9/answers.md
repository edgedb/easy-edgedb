# Chapter 9 Questions and Answers

#### 1. Why doesn't this insert work and how can it be fixed?

It needs to be a set instead of an array, so change the brackets to `{}` instead:

```edgeql
FOR castle IN {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
UNION (
  INSERT Castle {
    name := castle
  }
);
```

#### 2. How would you do the same insert while displaying the castle's name at the same time?

It looks like this:

```edgeql
SELECT (
  FOR castle IN {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
  UNION (
    INSERT Castle {
      name := castle
    }
  )
) { name };
```

#### 3. How would you change the `Vampire` type if all vampires needed a minimum strength of 10?

Since `strength` comes from `abstract type Person`, you would need to overload it and give it a constraint. The `Vampire` type would then look like this:

```sdl
type Vampire extending Person {
  multi link slaves -> MinorVampire;
  overloaded property strength {
    constraint min_value(10)
  }
}
```

#### 4. How would you update all the `Person` types to show that they died on September 11, 1887?

You would give them each a `last_appearance` like this:

```edgeql
UPDATE Person
SET {
  last_appearance := <cal::local_date>'1887-09-11'
};
```

#### 5. All the `Person` characters that have an `e` or an `a` in their name have been brought back to life. How would you update to do this?

You can just `UPDATE` by using `LIKE` on a set instead of a single letter:

```edgeql
UPDATE Person FILTER .name LIKE {'%a%', '%e%'}
SET {
  last_appearance := {}
};
```

And if you wanted to display the results at the same time to make sure, it would look like this:

```edgeql
SELECT (
  UPDATE Person FILTER .name ILIKE {'%a%', '%e%'}
  SET {
    last_appearance := {}
  }
) {
  name,
  last_appearance
};
```
