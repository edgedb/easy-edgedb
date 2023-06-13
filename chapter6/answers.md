# Chapter 6 Questions and Answers

#### 1. This select is incomplete. How would you complete it so that it says "Pleased to meet you, I'm " and then the NPC's name?

You can do it with concatenation using `++`:

```edgeql
select NPC {
  name,
  greeting := "Pleased to meet you. I'm " ++ .name ++ '.'
};
```

#### 2. How would you update Mina's `places_visited` to include Romania if she went to Castle Dracula for a visit?

Here is one way:

```edgeql
update Person filter .name = 'Mina Murray'
set {
  places_visited += (select Place filter .name = 'Romania')
};
```

You can of course go with `update NPC` and `select Country` if you prefer.

Also, here is the same thing using `with`:

```edgeql
with
  mina := (select NPC filter .name = 'Mina Murray'),
  romania := (select Country filter .name = 'Romania'),
update mina
set {
  places_visited += romania
};
```

#### 3. With the set `{'W', 'J', 'C'}`, how would you display all the `Person` types with a name that contains any of these capital letters?

It looks like this:

```edgeql
with letters := {'W', 'J', 'C'}
select Person {
  name
} filter .name like '%' ++ letters ++ '%';
```

And should display these characters we've inserted so far:

```
{
  Object {name: 'Vampire Woman 1'},
  Object {name: 'Vampire Woman 2'},
  Object {name: 'Vampire Woman 3'},
  Object {name: 'Jonathan Harker'},
  Object {name: 'Count Dracula'},
}
```

The key is that `like` takes a string, so you can concatenate `%` on the left and right with `++`.

#### 4. How would you display this same query as JSON?

Getting JSON output is super easy by casting with `<json>`, but where does it go? You can't put it in front of `select`, and `<json>Person` isn't an expression either, so this won't work:

```edgeql
with letters := {'W', 'J', 'C'}
select <json>Person {
  name
} filter .name like '%' ++ letters ++ '%';
```

You need to wrap the `select` in brackets, cast with `<json>` and then select that:

```edgeql
with letters := {'W', 'J', 'C'}
select <json>(
  select Person {
    name
  } filter .name like '%' ++ letters ++ '%'
);
```

Or you can use `with` to do this:

```edgeql
with
  letters := {'W', 'J', 'C'},
  P := (
    select Person filter .name like '%' ++ letters ++ '%'
  )
select <json>P { name };
```

So you're selecting the casted-to-JSON version of the result of `select Person`.

#### 5. How would you add ' the Great' to every Person type?

Easy, just update without `filter`:

```edgeql
update Person
set {
  name := .name ++ ' the Great'
};
```

Now their names are 'Vampire Woman 1 the Great', 'Mina Murray the Great', and so on.

**Bonus question**: to undo this, just set `name` to the same string minus the last 10 characters using `[0:-10]`:

```edgeql
update Person
set {
  name := .name[0:-10]
};
```
