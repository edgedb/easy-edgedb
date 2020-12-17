# Chapter 1 Questions and Answers

#### 1. Entering `SELECT my_name = 'Timothy' != 'Benjamin';` returns an error. Try adding one character to make it return `{true}`.

Answer: change `my_name =` to `my_name :=`. This will assign the string 'Timothy' to `my_name`, and then it will compare it to `'Benjamin'`, and return `{true}` because they are not equal.

---

#### 2. Try inserting a `City` called Constantinople, but now known as İstanbul.

```edgeql
INSERT City {
  name := 'Constantinople',
  modern_name := 'İstanbul' # Comma after here is fine if you want
};
```

---

#### 3. Try displaying all the names of the cities in the database.

This is all you need: `SELECT City.name;`

---

#### 4. Try selecting all the `City` types along with their `name` and `modern_name` properties, but change `.name` to say `old_name` and change `modern_name` to say `name_now`.

```edgeql
SELECT City {
  old_name := .name,
  name_now := .modern_name,
};
```

Of course, this will not change the name of the properties. `old_name` and `name_now` only show up in this one query.

---

#### 5. Will typing `SelecT City;` produce an error?

No, because keywords are case insensitive.
