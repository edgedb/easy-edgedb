# Chapter 7 Questions and Answers

#### 1. How would you select each City and the length of its name?

Easy, just use the `len()` function:

```edgeql
SELECT City {
  name,
  name_length := len(.name)
};
```

#### 2. How would you select each City and the length of its `name` minus the length of `modern_name` if `modern_name` exists, and 0 if `modern_name` does not exist?

For this we use our old friends `IF EXISTS` and `ELSE`:

```edgeql
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 0
};
```

#### 3. What if you wanted to write 'Modern name does not exist' instead of 0?

First of all, here's what we can't do:

```edgeql
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

You can probably guess why: items inside `length_difference` could be an `int64` sometimes, and `str` at other times, which is unacceptable. Here's the error:

```
error: operator 'std::IF' cannot be applied to operands of type 'std::int64', 'std::bool', 'std::str'
```

Fortunately, we can just cast the results to a string:

```edgeql
SELECT City {
  name,
  length_difference := <str>(len(.name) - len(.modern_name)) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

It should give this result:

```
{
  default::City {name: 'Munich', length_difference: 'Modern name does not exist'},
  default::City {name: 'Buda-Pesth', length_difference: '2'},
  default::City {name: 'Bistritz', length_difference: '0'},
  default::City {name: 'London', length_difference: 'Modern name does not exist'},
}
```

#### 4. How would you insert an NPC with the name 'NPC number 8' if for example there are already seven other NPCs?

It looks like this:

```edgeql
INSERT NPC {
  name := 'NPC number ' ++ <str>(count(DETACHED NPC) + 1)
};
```

`SELECT count(NPC)` on its own gives the number, but we are inserting an `NPC` at the same time so we need `DETACHED` to select the `NPC` type in general.

#### 5. How would you select only the `Person` types that have the shortest names?

First, here's how **not** to do it:

```edgeql
SELECT Person {
  name,
} FILTER len(.name) = min(len(Person.name));
```

This seems like it might work, but `min(len(Person.name))` is the minimum length of `Person` that we are selecting - in other words, one `Person`. The result: every Person type and their names show up.

Adding `DETACHED` solves it:

```edgeql
SELECT Person {
  name,
} FILTER len(.name) = min(len(DETACHED Person.name));
```

We could do the same with `WITH` as well, which is maybe a bit easier to read:

```edgeql
WITH minimum_length := min(len(DETACHED Person.name))
SELECT Person {
  name,
} FILTER len(.name) = minimum_length;
```
