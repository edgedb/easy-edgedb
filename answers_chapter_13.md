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

2.

3.

4.

5.
