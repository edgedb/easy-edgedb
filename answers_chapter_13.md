# Chapter 13 Questions and Answers

#### 1. How would you insert an `NPC` named 'Mr. Swales' who has visited the `City` called 'York', the `Country` called 'England', and the `OtherPlace` called 'Whitby Abbey'? Try it in a single insert.

Here it is, similar to the `Ship` insert we did:

```
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

```
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
      schema::Property {id: 70719b55-2031-11eb-b0e6-81bc5b4ee64b},
      schema::Property {id: 7075a94c-2031-11eb-9cb0-6d25e19e0e31},
    },
    links: {
      schema::Link {id: 70798bba-2031-11eb-b091-1d60d8428852},
      schema::Link {id: 70740220-2031-11eb-9bd0-f94fde12382f},
      schema::Link {id: 7076b0c7-2031-11eb-9cab-b7b889902e39},
    },
  },
}
```

Add `: {name}` in two places to make it fully readable:

```
SELECT (INTROSPECT Ship) {
  name,
  properties: {
    name
    },
  links: {
    name
    },
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
      schema::Link {name: 'crew'},
      schema::Link {name: '__type__'},
      schema::Link {name: 'sailors'},
    },
  },
}
```

#### 3. What would be the shortest way to see what links from the `Vampire` type?

Similar to how `SELECT Vampire.name` just gives all the names for the `Vampire` type (as opposed to `SELECT Vampire { name }`, you can do this:

``` 
SELECT (Introspect Vampire).links { name };
```

Here's the output:

```
{
  schema::Link {name: 'slaves'},
  schema::Link {name: '__type__'},
  schema::Link {name: 'lover'},
  schema::Link {name: 'places_visited'},
}
```

#### 4. What do you think the output of `SELECT DISTINCT {1, 2} + {1, 2};` will be?

Here's the output:

```
{2, 3, 3, 4}
```

You can see that `DISTINCT` works individually on each set, so `SELECT DISTINCT {1, 2} + {1, 2};` and `SELECT {1, 2} + {1, 2};` are the same. But if you were to write `SELECT {2, 2}` the output would be just `{2}`.

#### 5. What do you think the output of `SELECT DISTINCT {2, 2} + {2, 2};` will be?

The output will be `{2, 2}` because `DISTINCT` only works on the first set.

To get the output {2}, you can repeat the `DISTINCT`: `SELECT DISTINCT {2, 2} + DISTINCT {2, 2};`
