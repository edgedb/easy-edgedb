# Chapter 13 Questions and Answers

#### 1. How would you insert an `NPC` named 'Mr. Swales' who has visited the `City` called 'York', the `Country` called 'England', and the `OtherPlace` called 'Whitby Abbey'? Try it in a single insert.

Here it is, similar to the `Ship` insert we did:

```edgeql
INSERT NPC {
  name := 'Mr. Swales',
  places_visited := {
    (INSERT City {
      name := 'York'
    }),
    (INSERT Country {
      name := 'England'
    }),
    (INSERT OtherPlace {
      name := 'Whitby Abbey'
    }),
  }
};
```

#### 2. How readable is this introspect query?

This query:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links
};
```

is one third readable: `name` will actually show up as a real human-readable name. Here's the result:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 28e76c59-0209-11ec-adaa-85e1ecb99e47},
      schema::Property {id: 28e94e33-0209-11ec-9818-fb533a2c495f},
    },
    links: {
      schema::Link {id: 28e87ca8-0209-11ec-9ba8-71ef0b23db38},
      schema::Link {id: 28e8ee51-0209-11ec-b47e-8fd9b07debd3},
      schema::Link {id: 29176353-0209-11ec-a6c5-797987ef08b5},
    },
  },
}
```

Add `: {name}` in two places to make it fully readable:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {name},
  links: {name},
};
```

Now it gives this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'name'},
    },
    links: {
      schema::Link {name: '__type__'},
      schema::Link {name: 'crew'},
      schema::Link {name: 'sailors'},
    },
  },
}
```

#### 3. What would be the shortest way to see what links from the `Vampire` type?

Similar to how `SELECT Vampire.name` just gives all the names for the `Vampire` type (as opposed to `SELECT Vampire { name }`, you can do this:

```edgeql
SELECT (Introspect Vampire).links { name };
```

Here's the output:

```
{
  schema::Link {name: 'slaves'},
  schema::Link {name: 'lover'},
  schema::Link {name: 'places_visited'},
  schema::Link {name: '__type__'},
}
```

#### 4. What do you think the output of `SELECT DISTINCT {1, 2} + {1, 2};` will be?

Here's the output:

```
{2, 3, 3, 4}
```

You can see that `DISTINCT` works individually on each set, so `SELECT DISTINCT {1, 2} + {1, 2};` and `SELECT {1, 2} + {1, 2};` are the same. But if you were to write `SELECT DISTINCT {2, 2}` the output would be just `{2}`.

#### 5. What do you think the output of `SELECT DISTINCT {2, 2} + {2, 2};` will be?

The output will be `{4, 4}` because `DISTINCT` only works on the first set.

To get the output `{4}`, you can repeat the `DISTINCT`: `SELECT DISTINCT {2, 2} + DISTINCT {2, 2};`. Or you can wrap the whole thing like this: `SELECT DISTINCT({2, 2} + {2,2})`.
