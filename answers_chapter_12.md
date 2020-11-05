# Chapter 12 Questions and Answers

#### 1. Consider these two functions. Will EdgeDB accept the second one?

No, because the input signature for both of them is the same:

```
function gives_number(input: int64) -> int64
 using(input);
 
function gives_number(input: int64) -> int32
 using(<int32>input);
```

There is no way to know which of the two you want to use if both could take an `int64`.

Here's the error: `error: cannot create the `default::gives_number(input: std::int64)` function: a function with the same signature is already defined`

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
error: could not find a function variant make64
```

That's because `SELECT make64(8)` is giving it an `int64`, and it doesn't have a signature for that. You would need to cast with `SELECT make64(<int32>8);` (or `<int16>` to make it work.

#### 3. Will `SELECT {} ?? {3, 4} ?? {5, 6};` work?

No, because EdgeDB doesn't know what type `{}` is. But if you cast the `{}` to an `int64`:

```SELECT <int64>{} ?? {3, 4} ?? {5, 6};```

Then it will give the output `{3, 4}`.

#### 4. Will `SELECT <int64>{} ?? <int64>{} ?? {1, 2}` work?

Yes, with the output `{1, 2}`. If you continue to use `??`, it will keep going until it hits something that is not empty.

Knowing that, you can probably guess the output of this:

```
SELECT <int64>{} ?? <int64>{} ?? {1} ?? <int64>{} ?? {5};
```

The output is `{1}`, the first not empty set that it sees.
