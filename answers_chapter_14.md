# Chapter 14 Questions and Answers

#### 1. How would you display just the numbers for all the `Person` types? e.g. if there are 20 of them, displaying `1, 2, 3..., 18, 19, 20`.

One way to do it is with `enumerate()`. This is easy if we are just starting from 0, since `enumerate()` gives a tuple with two items. The first one is an `int64` so we select that:

```
SELECT enumerate(Person).0;
```

That will display: `{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19}`

Warning: selecting `.1` will generate an error since that is where the rest of the object type is, and there is no way to properly display that. But displaying a single property is fine, such as in this example:

```
SELECT enumerate(Person.strength).1;
```

So now how to display starting from 1? With a `FOR` loop:

```
WITH numbers := enumerate(Person).0,
 FOR number in {numbers}
 UNION(
 SELECT number + 1);
```

Don't forget to write `number in {numbers}` so that `numbers` becomes a set.

#### 2. Using a reverse lookup, how would you display 1) all the `Place` types (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

This is not too hard if you start it in steps, first with a filter to get all the `Place` types:

```
SELECT Place {
  name
  } FILTER .name LIKE '%o%';
```

And here they are:

```
{
  default::Country {name: 'Romania'},
  default::Country {name: 'Slovakia'},
  default::City {name: 'London'},
}
```

Now we'll add the reverse lookup to the same query, and call the computable `visitors`:

```
SELECT Place {
 name,
 visitors := .<places_visited[IS Person].name
 } FILTER .name LIKE '%o%';
```

Now we can see who visited:

```
{
  default::Country {name: 'Romania', visitors: {}},
  default::Country {name: 'Slovakia', visitors: {}},
  default::City {
    name: 'London',
    visitors: {
      'Lucy Westenra',
      'The innkeeper',
      'Mina Murray',
      'John Seward',
      'Quincey Morris',
      'Arthur Holmwood',
      'Renfield',
      'Abraham Van Helsing',
      'Emil Sinclair',
    },
  },
}
```

A clear victory for London as the most visited place! If you wanted, you could also add a `visitor_numbers := .<places_visited[IS Person].name` to it to get the number of visitors too.

3.

4.

5.
