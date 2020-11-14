# Chapter 15 Questions and Answers

#### 1. How would you create a type called Horse with a `required property name -> str` that can only be 'Horse'?

This is easy with `expression on` and using `__subject__.name` to refer to the name of the object we are inserting:

```
type Horse {
  required property name -> str;
  constraint expression on (__subject__.name = 'Horse');
};
```

#### 2. How would you make sure that `name` for type `PC` is always between 5 and 30 characters in length?

First of all, here is the type right now:

```
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

Since `name` comes from Person, we can overload it with this constraint. With `expression on` and `len` we can make sure that it's always greater than 4 and less than 31:

```
type NPC extending Person {
  overloaded property name {
    constraint expression on (len(__subject__) > 4 AND len(__subject__) < 31)
    }
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

Another option is just to use the `max_len_value()` and `min_len_value()` functions that we learned in this chapter.

#### 3. How would you make a function called `display_coffins` that pulls up all the `HasCoffins` types with more than 0 coffins?

Here's one way to do it:

```
function display_coffins() -> SET OF HasCoffins
  using(
    SELECT HasCoffins FILTER .coffins > 0
);
```

Note though that `HasCoffins` doesn't have properties like `name` so queries using it would look something like this:

```
SELECT display_coffins() {
  [IS City].name,
  [IS City].population,
};
```

4.

5.
