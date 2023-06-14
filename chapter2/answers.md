# Chapter 2 Questions and Answers

#### 1. Change the following `select` to display `{100}` by casting: `select '99' + '1'`;

You can cast with a numeric type, such as `select <int64>'99' + <int64>'1'`. Note that the types don't have to be the same, but EdgeDB will convert them automatically if you do this. You can see this with a query like `select <int64>'99' + <int16>'1' is int64;`, which returns `{true}`: it will convert the smaller `int16` to the larger `int64` if you add them together.

#### 2. Select all the `City` types that start with 'Mu' (case sensitive).

`select City filter .name like 'Mu%';` - don't forget the `%` so that it will match cities like Munich.

#### 3. Select the third letter (i.e. index number 2) of the name of every city.

`select NPC.name[2];` will do it.

#### 4. If some `NPC` objects might have a name of `''`, how would you find the third letter of the name of every `NPC`?

You can use a slice that goes from index 2 up to the next index (index 3). `select NPC.name[2:3];` will do it.

#### 5. How would you change the `Person` type to extend `HasAString`?

Just add `extending HasAString` to the type, which would now look like this:

```sdl
abstract type Person extending HasAString {
  required name: str;
  multi places_visited: City;
}
```

#### 6. This query only shows the id numbers of the places visited. How do you show their name?

Change it to this:

```edgeql
select Person {
  places_visited: {name}
};
```

Don't forget the `:` and remember that spacing doesn't matter, so you could write it this way too:

```edgeql
select Person {
  places_visited: {
    name
    }
};
```
