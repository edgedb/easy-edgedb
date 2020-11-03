# Chapter 7 Questions and Answers

#### 1. How would you select each City and the length of its name?

Easy, just use the `len()` function:

```
SELECT City {
  name,
  name_length := len(.name)
};
```

#### 2. How would you select each City and the length of its `name` minus the length of `modern_name` if `modern_name` exists, and 0 if `modern_name` does not exist?

For this we use our old friends `IF EXISTS` and `ELSE`:

```
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 0
};
```

#### 3. What if you wanted to write 'Modern name does not exist' instead of 0?

First of all, here's what we can't do:

```
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

You can probably guess why: items inside `length_difference` could be an `int64` sometimes, and `str` at other times, which is unacceptable.

Fortunately, we can just cast the results to a string:

```
SELECT City {
  name,
  length_difference := <str>(len(.name) - len(.modern_name)) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

It should give this result:

```
{
  Object {name: 'Munich', length_difference: 'Modern name does not exist'},
  Object {name: 'Buda-Pesth', length_difference: '2'},
  Object {name: 'Bistritz', length_difference: '0'},
  Object {name: 'London', length_difference: 'Modern name does not exist'},
}
```
