# Chapter 15 Questions and Answers

#### 1. How would you create a type called Horse with a `required property name -> str` that can only be 'Horse'?

This is easy with `expression on` and using `__subject__.name` to refer to the name of the object we are inserting:

```
type Horse {
  required property name -> str;
  constraint expression on (__subject__.name = 'Horse');
};
```

2.

3.

4.

5.
