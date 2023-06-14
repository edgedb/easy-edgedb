# Chapter 15 Questions and Answers

#### 1. How would you create a type called Horse with a `required name: str` that can only be 'Horse'?

This is easy with `expression on` and using `__subject__.name` to refer to the name of the object we are inserting:

```sdl
type Horse {
  required name: str;
  constraint expression on (__subject__.name = 'Horse');
};
```

#### 2. How would you let the user know that it needs to be called 'Horse'?

Just put it inside the `constraint` like this:

```sdl
type Horse {
  required name: str;
  constraint expression on (__subject__.name = 'Horse') {
    errmessage := 'All Horses must be named \'Horse\''
  }
};
```

#### 3. How would you make sure that `name` for type `PC` is always between 5 and 30 characters in length?

First of all, here is the type right now:

```sdl
type NPC extending Person {
  overloaded age {
    constraint max_value(120)
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

Since `name` comes from Person, we can overload it with this constraint. With `expression on` and `len` we can make sure that it's always greater than 4 and less than 31:

```sdl
type NPC extending Person {
  overloaded name {
    constraint expression on (len(__subject__) > 4 and len(__subject__) < 31)
  }
  overloaded age {
    constraint max_value(120)
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

Another option is just to use the `max_len_value()` and `min_len_value()` constraints that we learned in this chapter.

#### 4. How would you make a function called `display_coffins` that pulls up all the `HasCoffins` types with more than 0 coffins?

Here's one way to do it:

```sdl
function display_coffins() -> set of HasCoffins
  using(
    select HasCoffins filter .coffins > 0
  );
```

Note though that `HasCoffins` doesn't have properties like `name` so queries using it would look something like this:

```edgeql
select display_coffins() {
  [is City].name,
  [is City].population,
};
```

#### 5. How would you make it without touching the schema?

Easy, just put `create` in front of it!
