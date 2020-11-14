# Chapter 16 Questions and Answers

#### 1. How would you split all the `Person` names into two strings if they have two words, and ignore any that don't have exactly two words?

You can do that with `str_split` over empty spaces and then filtering by length:

```
WITH two_names := (SELECT str_split(Person.name, ' '))
  SELECT two_names FILTER len(two_names) = 2;
```

#### 2. How would you display all the `Person` names and where the string 'ma' is in their name?

The query is quite short:

```
SELECT (Person.name, find(Person.name, 'ma'));
```

Note that the first `Person.name` and the second `Person.name` are the same, which is why no Cartesian multiplication is used. However, if you changed the second one to `DETACHED Person.name` it would, and you would get well over 100 results.

3. 

4.

#### 5. How would you use `re_match_all()` to display all the `Person.name`s with `Crewman` in the name? e.g. Crewman 1, Crewman 2, etc.

You can do it by using the `.` wildcard to match anything:

```
SELECT re_match_all('Crewman .', Person.name);
```
