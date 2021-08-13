# Chapter 12 Questions and Answers

#### 1. Consider these two functions. Will EdgeDB accept the second one?

No, because the input signature for both of them is the same:

```sdl
function gives_number(input: int64) -> int64
  using(input);

function gives_number(input: int64) -> int32
  using(<int32>input);
```

There is no way to know which of the two you want to use if both could take an `int64`.

Here's the error:

```
error: cannot create the `default::gives_number(input: std::int64)` function: a function with the same signature is already defined
```

#### 2. How about these two functions? Will EdgeDB accept the second one?

Yes, they have different signatures because the input is different: one takes an `int16` and the other takes an `int32`. In fact, you can see both of them by using DESCRIBE. For example, `DESCRIBE FUNCTION make64 AS TEXT` gives the following:

```
{
  'function default::make64(input: std::int16) ->  std::int64 using (input);
   function default::make64(input: std::int32) ->  std::int64 using (input);',
}
```

Note however that `SELECT make64(8);` will actually produce an error! The error is:

```
error: function "make64(arg0: std::int64)" does not exist
  ┌─ query:1:8
  │
1 │ SELECT make64(8);
  │        ^^^^^^^^^ Did you want one of the following functions instead:
default::make64(input: std::int16)
default::make64(input: std::int32)
```

That's because `SELECT make64(8)` is giving it an `int64`, and it doesn't have a signature for that. You would need to cast with `SELECT make64(<int32>8);` (or `<int16>`) to make it work.

#### 3. Will `SELECT {} ?? {3, 4} ?? {5, 6};` work?

No, because EdgeDB doesn't know what type `{}` is. But if you cast the `{}` to an `int64`:

```edgeql
SELECT <int64>{} ?? {3, 4} ?? {5, 6};
```

Then it will give the output `{3, 4}`.

#### 4. Will `SELECT <int64>{} ?? <int64>{} ?? {1, 2}` work?

Yes, with the output `{1, 2}`. If you continue to use `??`, it will keep going until it hits something that is not empty.

Knowing that, you can probably guess the output of this:

```edgeql
SELECT <int64>{} ?? <int64>{} ?? {1} ?? <int64>{} ?? {5};
```

The output is `{1}`, the first not empty set that it sees.

#### 5. Trying to make a single string of everyone's name with `SELECT array_join(array_agg(Person.name));` isn't working. What's the problem?

The error message gives a hint:

`error: function "array_join(arg0: array<std::str>)" does not exist`

That means that the input that it received doesn't match any of its function signatures. And if you check [the function itself](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_join), you can see why: it needs a second string:

```sdl
std::array_join(array: array<str>, delimiter: str) -> str
```

So we can just change it to `SELECT array_join(array_agg(Person.name), ' ');` or `SELECT array_join(array_agg(Person.name), ' is awesome ');` or anything else with a second string and it works.
