# Chapter 4 Questions and Answers

1. This insert is not working.

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
