# Chapter 10 Questions and Answers

#### 1. Try inserting two `NPC` types in one insert.

It would look like this:

```edgeql
FOR person IN {('Jimmy the Bartender', '1887-09-10', '1887-09-11'), ('Some friend of Jonathan Harker', '1887-07-08', '1887-07-09')}
UNION (
  INSERT NPC {
    name := person.0,
    first_appearance := <cal::local_date>person.1,
    last_appearance := <cal::local_date>person.2
  }
);
```

#### 2. Here are two more `NPC`s to insert, except the last one has an empty set (she's not dead). What problem are we going to have?

The problem is that EdgeDB doesn't know what type the last `{}` is. You can see it by doing a quick `SELECT {('Dracula\'s Castle visitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05-08', {})}`:

The output is:

```
ERROR: QueryError: operator 'UNION' cannot be applied to operands of type 'tuple<std::str, std::str, std::str>' and 'tuple<std::str, std::str, anytype>'
  Hint: Consider using an explicit type cast or a conversion function.
```

A cast to `<str>{}` is an option, but we could do something more robust. First change the `{}` to an empty string, and then specify that `last_appearance` has to have a length of 10, and otherwise make it `<cal::local_date>{}`:

```edgeql
FOR person IN {('Dracula\'s Castle visitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05-08', '')}
UNION(
  INSERT NPC {
    name := person.0,
    first_appearance := <cal::local_date>person.1 IF len(person.1) = 10 ELSE <cal::local_date>{},
    last_appearance := <cal::local_date>person.2 IF len(person.2) = 10 ELSE <cal::local_date>{}
  }
);
```

#### 3. How would you order the `Person` types by last letter of their names?

Just order by doing a slice of the final letter:

```edgeql
SELECT Person {
  name
} ORDER BY .name[-1];
```

#### 4. Try inserting an `NPC` with the name `''`. Now how would you do the same query in question 3?

If we try to do the same query as above, we get the following error:

```
ERROR: InvalidValueError: string index -1 is out of bounds
```

No problem though. One way is to filter out anything with a length under 1:

```edgeql
SELECT Person {
  name
} FILTER len(.name) > 0 ORDER BY .name[-1];
```

Or if you don't want to filter it out, you could just add a dummy character for anything without a length of at least 1:

```edgeql
SELECT Person {
  name := ' ' ++ .name IF len(.name) = 0 ELSE .name
} ORDER BY .name[-1];
```

#### 5. Dr. Van Helsing has a list of MinorVampires with their names and strengths. We already have some MinorVampires in the database. How would you INSERT them while making sure to UPDATE if the object is already there?

You can do it with `UNLESS CONFLICT ON` and `ELSE`. An insert for one named Carmilla then would look like this:

```edgeql
INSERT MinorVampire {
  name := 'Carmilla',
  strength := 10,
} UNLESS CONFLICT ON .name
ELSE (
  UPDATE MinorVampire
  SET {
    name := 'Carmilla',
    strength := 10,
  }
);
```

Then even if his data has a conflicting name (like 'Woman 1'), it will still update with their correct strength instead of just giving up:

```edgeql
INSERT MinorVampire {
  name := 'Woman 1',
  strength := 7,
} UNLESS CONFLICT ON .name
ELSE (
  UPDATE MinorVampire
  SET {
    name := 'Woman 1',
    strength := 7,
  }
);
```
