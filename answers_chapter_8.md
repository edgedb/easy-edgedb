# Chapter 8 Questions and Answers

#### 1. How would you select all the `Place` types and their names, plus the `door` property if it's a `Castle`?

Like this:

```
SELECT Place {
  name,
  [IS Castle].doors
};
```

Without `[IS Castle]`, it will give an error.

#### 2. How would you select `Place` types with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

```
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
};
```

#### 3. How would you do the same but only showing the results of `City` and `Country` types?

This question is because Question 2 gives this result:

```
{
  Object {city_name: {}, country_name: {}},
  Object {city_name: 'Munich', country_name: {}},
  Object {city_name: 'Buda-Pesth', country_name: {}},
  Object {city_name: 'Bistritz', country_name: {}},
  Object {city_name: 'London', country_name: {}},
  Object {city_name: {}, country_name: 'Romania'},
  Object {city_name: {}, country_name: 'Slovakia'},
  Object {city_name: {}, country_name: {}},
}
```

The `Object {city_name: {}, country_name: {}},` lines are not useful to us to we'll filter them out with `EXISTS`:

```
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
} FILTER EXISTS .city_name OR EXISTS .country_name;
```

#### 4. How would you display all the `Person` types that don't have `lover`s, with their names and their type names?

To get all these single people, names and object types, just do this:

```
SELECT Person {
  name,
  __type__: {
    name
    }
} FILTER NOT EXISTS .lover;
```

Don't forget `name` after type! It won't make an error but the type name will be something like this and not very helpful: `__type__: Object {id: 20ef52ae-1d97-11eb-8cb6-0de731b01cc9}`
