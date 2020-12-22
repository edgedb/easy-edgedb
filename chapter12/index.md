# Chapter 12 - From bad to worse

There is no good news for our heroes this chapter:

> Dracula continues to break into Lucy's room every time people don't listen to Van Helsing, and every time the men give their blood to save her. Dracula always turns into a cloud to sneak in, drinks her blood and sneaks away before morning. Lucy is getting so weak that it's a surprise that she's still alive. Meanwhile, Renfield breaks out of his cell and attacks Dr. Seward with a knife. He cuts him with it, and the moment he sees Dr. Seward's blood he stops and tries to drink it, repeating: “The blood is the life! The blood is the life!”. The asylum security men take Renfield away and Dr. Seward is left confused and trying to understand him. He thinks there is a connection between him and the other events. That night, a wolf controlled by Dracula breaks the windows of Lucy's room and Dracula is able to get in again...

But there is good news for us, because we are going to keep learning about Cartesian products, plus how to overload a function.

## Overloading functions

Last chapter, we used the `fight()` function for some characters, but most only have `{}` for the `strength` property. That's why the Innkeeper defeated Dracula, which is obviously not what would really happen.

Jonathan Harker is just a human but is still quite strong. We'll give him a strength of 5. We'll treat that as the maximum strength for a human, except Renfield who is a bit unique. Every other human should have a strength between 1 and 5. EdgeDB has a random function called `std::rand()` that gives a `float64` in between 0.0 and 1.0. There is another function called [round()](https://www.edgedb.com/docs/edgeql/funcops/generic/#function::std::round) that rounds numbers, so we'll use that too, and finally cast it to an `<int16>`. Our input looks like this:

```edgeql
SELECT <int16>round((random() * 5));
```

So now we'll use this to update our Person types and give them all a random strength.

```edgeql
WITH random_5 := (SELECT <int16>round((random() * 5)))
 # WITH isn't necessary - just making the query prettier

UPDATE Person
FILTER NOT EXISTS .strength
SET {
  strength := random_5
};
```

And we'll make sure Count Dracula gets 20 strength, because he's Dracula:

```edgeql
UPDATE Vampire
FILTER .name = 'Count Dracula'
SET {
  strength := 20
};
```

Now let's `SELECT Person.strength;` and see if it works:

```
{3, 3, 3, 2, 3, 2, 2, 2, 3, 3, 3, 3, 4, 1, 5, 10, 4, 4, 20, 4, 4, 4, 4}
```

Looks like it worked.

So now let's overload the `fight()` function. Right now it only works for one `Person` vs. another `Person`, but in the book all the characters get together to try to defeat Dracula. We'll need to overload the function so that more than one character can work together to fight. There are a lot of ways to do it, but we'll choose a simple one:

```sdl
function fight(names: str, one: int16, two: Person) -> str
  using (
    SELECT names ++ ' win!' IF one > two.strength ELSE two.name ++ ' wins!'
  );
```

Note that overloading only works if the function signature is different. Here are the two signatures we have now for comparison:

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: Person) -> str
```

If we tried to overload it with an input of `(Person, Person)`, it wouldn't work because it's the same. That's because EdgeDB uses the input we give it to know which form of the function to use.

So now it's the same function name, but we enter the names of the people together, their strength together, and then the `Person` they are fighting.

Now Jonathan and Renfield are going to try to fight Dracula together. Good luck!

```edgeql
WITH
  jon_and_ren_strength := <int16>(
    SELECT sum(
      (SELECT NPC FILTER .name IN {'Jonathan Harker', 'Renfield'}).strength
    )
  ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

SELECT fight('Jon and Ren', jon_and_ren_strength, dracula);
```

So did they...

```
{'Count Dracula wins!'}
```

No, they didn't win. How about four people?

```edgeql
WITH
  four_people_strength := <int16>(
    SELECT sum(
      (
        SELECT NPC
        FILTER .name IN {'Jonathan Harker', 'Renfield', 'Arthur Holmwood', 'The innkeeper'}
      ).strength
    )
  ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

  SELECT fight('The four people', four_people_strength, dracula);
```

Much better:

```
{'The four people win!'}
```

So that's how function overloading works - you can create functions with the same name as long as the signature is different.

You see overloading in a lot of existing functions, such as [sum](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::sum) which takes in all numeric types and returns the sum. [std::to_datetime](https://www.edgedb.com/docs/edgeql/funcops/datetime#function::std::to_datetime) has even more interesting overloading with all sorts of inputs to create a `datetime`.

`fight()` was pretty fun to make, but that sort of function is better done on the gaming side. So let's make a function that we might actually use. Since EdgeQL is a query language, the most useful functions are usually ones that make queries shorter.

Here is a simple one that tells us if a `Person` type has visited a `Place` or not:

```sdl
function visited(person: str, city: str) -> bool
  using (
    WITH person := (SELECT Person FILTER .name = person LIMIT 1),
    SELECT city IN person.places_visited.name
  );
```

Now our queries are much faster:

```edgeql-repl
edgedb> SELECT visited('Mina Murray', 'London');
{true}
edgedb> SELECT visited('Mina Murray', 'Bistritz');
{false}
```

Thanks to the function, even more complicated queries are still quite readable:

```edgeql
SELECT(
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' ++ <str>visited('Jonathan Harker', 'Romania')
);
```

This prints `{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`.

The documentation for creating functions [is here](https://www.edgedb.com/docs/edgeql/ddl/functions#create-function). You can see that you can create them with SDL or DDL but there is not much difference between the two. In fact, they are so similar that the only difference is the word `CREATE` that DDL needs. In other words, just add `CREATE` to make a function without touching the schema. For example, here's a function that just says hi:

```sdl
function say_hi() -> str
  using ('hi');
```

If you want to create it right now, just do this:

```edgeql
CREATE FUNCTION say_hi() -> str
  USING ('hi');
```

(or with lowercase letters, it doesn't matter)

You'll see more or less the same thing when you ask to `DESCRIBE FUNCTION say_hi`:

```
{'CREATE FUNCTION default::say_hi() ->  std::str USING (\'hi\');'}
```

## Deleting (dropping) functions

You can delete a function with the `DROP` keyword and the function signature. You only have to specify the input though, because the input is all that EdgeDB looks at when identifying a function. So in the case of our two `fight()` functions:

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: Person) -> str
```

You would delete them with `DROP fight(one: Person, two: Person)` and `DROP fight(names: str, one: int16, two: Person)`. The `-> str` part isn't needed.

## More about Cartesian products - the coalescing operator

Now let's learn more about Cartesian products in EdgeDB. You might be surprised to see that even a single `{}` input always results in an output of `{}`, but this is the way that Cartesian products work. Remember, a `{}` has a length of 0 and anything multiplied by 0 is also 0. For example, let's try to add the names of places that start with b and those that start with f.

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name;
```

The output is....maybe unexpected if you didn't read the previous paragraph.

```
{}
```

!! It's an empty set. But a search for places that start with b gives us `{'Buda-Pesth', 'Bistritz'}`. Let's see if the same works when we concatenate with `++` as well.

```edgeql
SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
```

So that should give a `{}`. The output is...

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast or a conversion function.
```

Another surprise! This is an important point though: EdgeDB requires a cast for an empty set, because it won't try to guess at what type it is. There's no way to guess the type of an empty set if all we give it is `{}`, so EdgeDB won't try. You can probably guess that the same is true for array constructors too, so `SELECT [];` returns an error: `QueryError: expression returns value of indeterminate type`.

Okay, one more time, this time making sure that the `{}` empty set is of type `str`:

```edgeql-repl
edgedb> SELECT {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
```

Good, so we have manually confirmed that using `{}` with another set always returns `{}`. But what if we want to:

- Concatenate the two strings if they exist, and
- Return what we have if one is an empty set?

In other words, how to add `{'Buda-Peth', 'Bistritz'}` to another set and return the original `{'Buda-Peth', 'Bistritz'}` if the second is empty?

To do that we can use the so-called [coalescing operator](https://www.edgedb.com/docs/edgeql/funcops/set#operator::COALESCE), which is written `??`. The explanation for the operator is nice and simple:

`Evaluate to A for non-empty A, otherwise evaluate to B.`

So if the item on the left is not empty it will return that, and otherwise it will return the one on the right.

Here is a quick example:

```edgeql-repl
edgedb> SELECT {'Count Dracula is now in Whitby'} ?? <str>{};
```

Because we used `??` instead of `++`, the result is `{'Count Dracula is now in Whitby'}` and not `{}`.

So let's get back to our original query, this time with the coalescing operator:

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name
  IF EXISTS b_places.name AND EXISTS f_places.name
  ELSE b_places.name ?? f_places.name;
```

This returns:

```
{'Buda-Pesth', 'Bistritz'}
```

That's better.

But now back to Cartesian products. Remember, when we add or concatenate sets we are working with _every item in each set_ separately. So if we change the query to search for places that start with b (Buda-Pesth and Bistritz) and m (Munich):

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     m_places := (SELECT Place FILTER Place.name ILIKE 'm%'),
SELECT b_places.name ++ ' ' ++ m_places.name
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

Then we'll get this result:

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

instead of something like 'Buda-Peth, Bistritz, Munich'.

To get the output that we want, we can use two more functions:

- First [array_agg](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_agg), which turns the set into an array.
- Next, [array_join](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_join) to turn the array into a string.

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     m_places := (SELECT Place FILTER Place.name ILIKE 'm%'),
SELECT array_join(array_agg(b_places.name), ', ') ++ ', ' ++
  array_join(array_agg(m_places.name), ', ')
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

Finally! The output is `{'Buda-Pesth, Bistritz, Munich'}`. Now with this more robust query we can use it on anything and don't need to worry about getting {} if we choose a letter like x. Let's look at every place that contains k or e:

```edgeql
WITH has_k := (SELECT Place FILTER Place.name ILIKE '%k%'),
     has_e := (SELECT Place FILTER Place.name ILIKE '%e%'),
SELECT array_join(array_agg(has_k.name), ', ') ++ ', ' ++
  array_join(array_agg(has_e.name), ', ')
  IF EXISTS has_k.name AND EXISTS has_e.name
  ELSE has_k.name ?? has_e.name;
```

This gives us the result:

```
{'Slovakia, Buda-Pesth, Castle Dracula'}
```

Similarly, you can use `?=` instead of `=` and `?!=` instead of `!=` when doing comparisons if you think one side might be an empty set. So then you can write a query like this:

```edgeql
WITH cities1 := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     cities2 := <str>{}, # Don't forget to cast to <str>
SELECT cities1 ?= cities2;
```

and get the output

```
{
  false,
  false,
  false
}
```

instead of `{}` for the whole thing. Also, two empty sets are treated as equal if you use `?=`. So this query:

```edgeql
SELECT Vampire.lover.name ?= Crewman.name;
```

will return `{true}`. (Because Dracula has no lover and the Crewmen have no names so both sides return empty sets of type `str`.)

[Here is all our code so far up to Chapter 12.](code.md)

## Time to practice

<!-- quiz-start -->

1. Consider these two functions. Will EdgeDB accept the second one?

   First function:

   ```sdl
   function gives_number(input: int64) -> int64
     using(input);
   ```

   Second function:

   ```sdl
   function gives_number(input: int64) -> int32
     using(<int32>input);
   ```

2. How about these two functions? Will EdgeDB accept the second one?

   First function:

   ```sdl
   function make64(input: int16) -> int64
     using(input);
   ```

   Second function:

   ```sdl
   function make64(input: int32) -> int64
     using(input);
   ```

3. Will `SELECT {} ?? {3, 4} ?? {5, 6};` work?

4. Will `SELECT <int64>{} ?? <int64>{} ?? {1, 2}` work?

5. Trying to make a single string of everyone's name with `SELECT array_join(array_agg(Person.name));` isn't working. What's the problem?

[See the answers here.](answers.md)

<!-- quiz-end -->

Up next in Chapter 13: [One of the men gives Lucy his blood again to try to save her. Will it be enough?](../chapter13/index.md)
