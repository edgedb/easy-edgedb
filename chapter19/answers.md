# Chapter 19 Questions and Answers

#### 1. How would you display all the `City` names and the names of the `Region` they are in?

This is not too hard for us now, just a computed backlink. We'll also put a filter on to only show `City` objects that are inside a `Region`:

```edgeql
select City {
  name,
  region_name := .<cities[is Region].name
} filter exists .region_name;
```

#### 2. How about the `City` names plus the names of the `Region` and the name of the `Country` they are in?

This is a similar query except that we need to go back two links this time.

In the same way that `.<cities[is Region].name` means "the name of the `Region` connected via a link called `cities`", now it looks like this:

`.<cities[is Region].<regions[is Country].name`

In other words, "the name of the `Country` connected via a link called `regions` to the type called `Region` connected to the city via a link called `cities`.

```edgeql
select City {
  name,
  region_name := .<cities[is Region].name,
  country_name := .<cities[is Region].<regions[is Country].name
} filter exists .country_name;
```

That gives us a nice output going from the `City` to the `Country` level:

```
{
  default::City {name: 'Dresden', region_name: {'Saxony'}, country_name: {'Germany'}},
  default::City {name: 'Leipzig', region_name: {'Saxony'}, country_name: {'Germany'}},
  default::City {name: 'Darmstadt', region_name: {'Hesse'}, country_name: {'Germany'}},
  default::City {name: 'Mainz', region_name: {'Hesse'}, country_name: {'Germany'}},
  default::City {name: 'Berlin', region_name: {'Prussia'}, country_name: {'Germany'}},
  default::City {name: 'Königsberg', region_name: {'Prussia'}, country_name: {'Germany'}},
}
```
