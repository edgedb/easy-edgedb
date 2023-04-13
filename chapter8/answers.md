# Chapter 8 Questions and Answers

#### 1. How would you select all the `Place` types and their names, plus the `doors` property if it's a `Castle`?

Like this:

```edgeql
SELECT Place {
  name,
  [IS Castle].doors
};
```

Without `[IS Castle]`, it will give an error.

#### 2. How would you select `Place` types with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

```edgeql
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
};
```

#### 3. How would you do the same but only showing the results of `City` and `Country` types?

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

The `Object {city_name: {}, country_name: {}},` lines are not useful to us to we'll filter them out with `EXISTS`:

```edgeql
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
} FILTER EXISTS .city_name OR EXISTS .country_name;
```

One other way to filter is to use `FILTER Place IS City | Country`. You might be familiar with `|` in other programming languages. In EdgeDB this is called the type union operator, and you'll learn more about it in Chapter 13.

#### 4. How would you display all the `Person` types that don't have `lover`s, with their names and their type names?

To get all these single people, names and object types, just do this:

```edgeql
SELECT Person {
  name,
  __type__: {
    name
  }
} FILTER NOT EXISTS .lover;
```

Don't forget `name` after type! It won't make an error but the type name will be something like this and not very helpful: `__type__: Object {id: 20ef52ae-1d97-11eb-8cb6-0de731b01cc9}`

#### 5. What needs to be fixed in this query? Hint: two things definitely need to be fixed, while one more should probably be changed to make it more readable.

The two parts that need to be fixed are: 1) a `,` after `name` and a `.` after `[IS Castle]`:

```edgeql
SELECT Place {
  __type__,
  name,
  [IS Castle].doors
};
```

The part that _should_ be fixed: we should probably put `name` inside `__type__` so that it's readable to us (instead of `__type__: Object {id: e0a9ab38-1e6e-11eb-9497-5bb5357741af}`). It looks like this:

```edgeql
SELECT Place {
  __type__: {
    name
  },
  name,
  [IS Castle].doors
};
```
