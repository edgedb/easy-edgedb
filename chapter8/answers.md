# Chapter 8 Questions and Answers

#### 1. How would you select all the `Place` type objects and their names, plus the `doors` property if it's a `Castle`?

Like this:

```edgeql
select Place {
  name,
  [is Castle].doors
};
```

Without `[is Castle]`, it will give an error.

#### 2. How would you select `Place` type objects with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

```edgeql
select Place {
  city_name := [is City].name,
  country_name := [is Country].name
};
```

#### 3. How would you do the same but only showing the results of `City` and `Country` objects?

This question is because Question 2 gives this result:

```
{
  default::City {city_name: 'Munich', country_name: {}},
  default::City {city_name: 'Buda-Pesth', country_name: {}},
  default::City {city_name: 'Bistritz', country_name: {}},
  default::City {city_name: 'London', country_name: {}},
  default::Castle {city_name: {}, country_name: {}},
  default::Country {city_name: {}, country_name: 'Hungary'},
  default::Country {city_name: {}, country_name: 'Romania'},
  default::Country {city_name: {}, country_name: 'France'},
  default::Country {city_name: {}, country_name: 'Slovakia'},
}
```

The `Object {city_name: {}, country_name: {}},` lines are not useful to us to we'll filter them out with `exists`:

```edgeql
select Place {
  city_name := [is City].name,
  country_name := [is Country].name
} filter exists .city_name or exists .country_name;
```

One other way to filter is to use `filter Place is City | Country`. You might be familiar with `|` in other programming languages. In EdgeDB this is called the type union operator, and you'll learn more about it in Chapter 13.

#### 4. How would you display all the `Person` objects that don't have `lover`s, with their names and their type names?

To get all these single people, names and object types, just do this:

```edgeql
select Person {
  name,
  __type__: {
    name
  }
} filter not exists .lover;
```

Don't forget `name` after type! It won't return an error without it but the type name will be something like this and not very helpful: `__type__: Object {id: 20ef52ae-1d97-11eb-8cb6-0de731b01cc9}`

#### 5. What needs to be fixed in this query? Hint: two things definitely need to be fixed, while one more should probably be changed to make it more readable.

The two parts that need to be fixed are: 

* A comma after `name`, and
* A period after `[is Castle]`:

```edgeql
select Place {
  __type__,
  name,
  [is Castle].doors
};
```

The part that _should_ be fixed: we should probably put `name` inside `__type__` so that it's readable to us (instead of `__type__: Object {id: e0a9ab38-1e6e-11eb-9497-5bb5357741af}`). It looks like this:

```edgeql
select Place {
  __type__: {
    name
  },
  name,
  [is Castle].doors
};
```
