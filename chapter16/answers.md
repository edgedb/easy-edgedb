# Chapter 16 Questions and Answers

#### 1. How would you split all the `Person` names into two strings if they have two words, and ignore any that don't have exactly two words?

You can do that with `str_split` over empty spaces and then filtering by length:

```edgeql
with two_names := (select str_split(Person.name, ' '))
select two_names filter len(two_names) = 2;
```

#### 2. How would you display all the `Person` names and where the string 'ma' is in their name?

The query is quite short:

```edgeql
select (Person.name, find(Person.name, 'ma'));
```

Note that the first `Person.name` and the second `Person.name` are the same, which is why no Cartesian multiplication is used. However, if you changed the second one to `detached Person.name` it would, and you would get well over 100 results.

#### 3. How would you index on the `pen_name` property for type Person?

Fortunately, you can do it just by using `index on (.pen_name)`.

But if you felt like it for some reason, you could specify it as an expression like this:

```sdl
index on (.name ++ ' ' ++ .degrees if exists .degrees else .name)
```

#### 4. How would you display the name of every `Person` in uppercase followed by a space and then the same name in lowercase?

Here is perhaps the easiest way to do it - just use the `str_upper` and `str_lower` methods to do the work and add a space with `++ ' '`:

```edgeql
select str_upper(Person.name) ++ ' ' ++ str_lower(Person.name);
```

#### 5. How would you use `re_match_all()` to display all the `Person.name`s with `Crewman` in the name? e.g. Crewman 1, Crewman 2, etc.

You can do it by using the `.` wildcard to match anything:

```edgeql
select re_match_all('Crewman .', Person.name);
```
