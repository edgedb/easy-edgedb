#### 1. This query is trying to display every `NPC` along with the `name` plus every `City` type for each `NPC`, but it's giving an error. What is it missing?

All it needs is brackets around the `SELECT` - remember, putting it in brackets "captures" the output so it can be used.

```
SELECT NPC {
  name,
  cities := (SELECT City.name)
};
```

#### 2. If the `City` type needed a required property called `population`, what would it look like? What type would 'population' be?

Right now `City` is just extending `Place`:

```
type City extending Place;
```

So it would need a `required property` for population:

```
type City extending Place {
  required property population -> int32
};
```

`int16` only goes up to 32,767 while both `int32` and `int64` are large enough, but `int32`'s maximum of 2,147,483,647 is certainly enough for a city.

#### 3. This query wants to display `name` twice for some reason but is giving an error. Can you think of a way to do it?

You can access `property name` twice by giving it a different name the second time. Let's call it name2:

```
SELECT Person {
  name,
  name2 := .name
};
```

#### 4. People keep trying to make characters with negative ages. Can you think of a constraint that can stop this?

We haven't seen this constraint but it's easy to guess: with `min_value()` you can do it. With these two constraints, HumanAge must be between 0 and 120:

```
scalar type HumanAge extending int16 {
  constraint max_value(120);
  constraint min_value(0);
}
```

#### 5. Can you insert a HumanAge type?

No, because it's a scalar type and not an object - a `HumanAge` would just be an `int16` connected to nothing.

But of course, you can select one by casting:

```
SELECT <HumanAge>16;
```

This gives `{16}`.

You can also see that it is just a different name for an `int16` by trying the following:

```
SELECT <HumanAge>16 IS int16;
SELECT <HumanAge>16 IS HumanAge;
```

Both of these return `{true}`.
