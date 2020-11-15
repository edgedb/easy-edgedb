# Chapter 19 Questions and Answers

#### 1. How would you display all the `City` names and the names of the `Region` they are in?

This is not too hard for us now, just a reverse lookup:

```
SELECT City {
  name,
  region_name := .<cities[IS Region].name
};
```

#### 2. How about the `City` names plus the names of the `Region` and the name of the `Country` they are in?

This is a similar query except that we need to go back two links this time.

In the same way that `.<cities[IS Region].name` means "the name of the `Region` connected via a link called `cities`", now it looks like this:

`.<cities[IS Region].<regions[IS Country].name`

In other words, "the name of the `Country` connected via a link called `regions` to the type called `Region` connected to the city via a link called `cities`.

```
SELECT City {
  name,
  region_name := .<cities[IS Region].name,
  country_name := .<cities[IS Region].<regions[IS Country].name
};
```


3.

4.

5.
