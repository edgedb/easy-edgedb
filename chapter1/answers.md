# Chapter 1 Questions and Answers

#### 1. Entering the code below returns an error. Try adding one character to make it return `{true}`.

Answer: change `with my_name = 'Timothy'` to `with my_name := 'Timothy'`. This will assign the string 'Timothy' to `my_name` (instead of trying to compare 'Timothy' to `my_name` which doesn't exist). Then it will compare it to `'Benjamin'` and return `{true}` because they are not equal.

---

#### 2. Try inserting a `City` called Constantinople, but now known as İstanbul.

```edgeql
insert City {
  name := 'Constantinople',
  modern_name := 'İstanbul' # Comma after here is fine if you want
};
```

---

#### 3. Try displaying all the names of the cities in the database.

This is all you need: `select City.name;`

---

#### 4. Try selecting all the `City` types along with their `name` and `modern_name` properties, but change `.name` to say `old_name` and change `modern_name` to say `name_now`.

```edgeql
select City {
  old_name := .name,
  name_now := .modern_name,
};
```

Of course, this will not change the name of the properties. `old_name` and `name_now` only show up in this one query.

---

#### 5. Will typing `SelecT City;` produce an error?

No, because keywords are case insensitive.
