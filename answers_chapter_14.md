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

2.

3.

4.

5.
