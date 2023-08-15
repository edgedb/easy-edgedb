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

#### 3. How would you make sure that `name` for type `NPC` is always between 5 and 30 characters in length?

First of all, here is the type right now:

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120)
  }
}
```

Since `name` comes from Person, we can overload it with this constraint. With `expression on` and `len` we can make sure that it's always greater than 4 and less than 31:

```sdl
type NPC extending Person {
  overloaded name: str {
    constraint expression on (len(__subject__) > 4 and len(__subject__) < 31)
  }
  overloaded age: int16 {
    constraint max_value(120)
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

#### 5. How would you give the `Place` type a backlink to every `Person` type that visited it?

The `Person` type has a link called `places_visited`, so we can use this to create a backlink on `Place`. It looks like this:

```sdl
link visitors := .<places_visited[is Person];
```

With this backlink in place we could do a query like the following to see who has visited Munich:

```edgeql
select Place { visitors: {*}} filter .name = 'Munich';
```

The query should turn up Emil Sinclair the `PC`, Jonathan Harker, and Lord Billy. Feel free to keep the backlink in your schema if you like as it won't make a difference for the rest of the book.