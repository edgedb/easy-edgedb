# Chapter 6 Questions and Answers

#### 1. This select is incomplete. How would you complete it so that it says "Pleased to meet you, I'm " and then the NPC's name?

You can do it with concatenation using `++`:

```
SELECT NPC {
  name,
  greeting := "Pleased to meet you, I'm " ++ .name
};
```

#### 2. How would you update Mina's `places_visited` to include Romania if she went to Castle Dracula for a visit?

Here is one way:

```
UPDATE Person FILTER .name = 'Mina Murray'
  SET {
    places_visited += (SELECT Place FILTER .name = 'Romania')
};
```

You can of course go with `UPDATE NPC` and `SELECT City` if you prefer.

Also, here is the same thing using `WITH`:

```
WITH 
  mina := (SELECT NPC FILTER .name = 'Mina Murray'),
  romania := (SELECT Country FILTER .name = 'Romania'),
UPDATE mina
SET {
  places_visited += romania
};
```

#### 3. With the set `{'W', 'J', 'C'}`, how would you display all the `Person` types with a name that contains any of these capital letters?

It looks like this:

```
WITH letters := {'W', 'J', 'C'}
  SELECT Person {
   name
} FILTER .name LIKE '%' ++ letters ++ '%';
```

And should display these characters we've inserted so far:

```
{
  Object {name: 'Woman 1'},
  Object {name: 'Woman 2'},
  Object {name: 'Woman 3'},
  Object {name: 'Jonathan Harker'},
  Object {name: 'Count Dracula'},
}
```

The key is that `LIKE` takes a string, so you can concatenate `%` on the left and right with `++`.

#### 4. How would you display this same query as JSON?

Getting JSON output is super easy by casting with `<json>`, but where does it go? You can't put it in front of `SELECT`, and `<json>Person` isn't an expression either, so this won't work:

```
WITH letters := {'W', 'J', 'C'}
  SELECT <json>Person {
    name
} FILTER .name LIKE '%' ++ letters ++ '%';
```

You need to wrap the SELECT in brackets, cast with `<json>` and then SELECT that:

```
WITH letters := {'W', 'J', 'C'}
  SELECT <json>(SELECT Person {
    name
} FILTER .name LIKE '%' ++ letters ++ '%');
```

So you're selecting the casted-to-JSON version of the result of `SELECT Person`.

#### 5. How would you add ' the Great' to every Person type?

Easy, just update without `FILTER`:

```
UPDATE Person
  SET {
    name := .name ++ ' the Great'
};
```

Now their names are 'Woman 1 the Great', 'Mina Murray the Great', and so on.

**Bonus question**: to undo this, just set `name` to the same string minus the last 10 characters using `[0:-10]`:

```
UPDATE Person
  SET {
    name := .name[0:-10]
};
```
