---
tags: Overloading Functions, Coalescing
---

# Chapter 12 - From bad to worse

There is no good news for our heroes this chapter:

> Dracula continues to break into Lucy's room every time people don't listen to Van Helsing, and every time the men give their blood to save her. Dracula always turns into a cloud to sneak in, drinks her blood and sneaks away before morning. Lucy is getting so weak that it's a surprise that she's still alive.
>Meanwhile, Renfield breaks out of his cell and attacks Dr. Seward with a knife, causing him to bleed. The moment Renfield sees Dr. Seward's blood he stops and tries to drink it, repeating: “The blood is the life! The blood is the life!”. The asylum security men take Renfield away and Dr. Seward is left confused and trying to understand him. Dr. Seward thinks there is a connection between him and the other events.
>That night, a wolf controlled by Dracula breaks the windows of Lucy's room and Dracula is able to get in again...

But there is good news for us, because we are going to keep learning about Cartesian products, plus how to overload a function.

## Overloading functions

Last chapter, we gave every character a strength of 5 and used the `fight()` function for a few of them. That's why the Innkeeper defeated Dracula, which is obviously not what would really happen. We should give Dracula a more realistic strength, and randomize some of the strength values for some of our characters.

Jonathan Harker is just a human but is still quite strong. We'll let him keep his strength of 5. We'll treat that as the maximum strength for a human, except Renfield who is a bit unique - we gave him a strength of 10 when we inserted him. And we'll make sure Count Dracula gets 20 strength, because he's Dracula. So we only have to update Count Dracula's strength:

```edgeql
update Vampire
filter .name = 'Count Dracula'
set {
  strength := 20
};
```

Every other human should have a strength between 1 and 5. EdgeDB has a random function called {eql:func}`docs:std::random` that gives a `float64` in between 0.0 and 1.0. There is another function called {eql:func}`docs:std::round` that rounds numbers, so we'll use that too, and finally cast it to an `<int16>`. Give this a try a few times to see the `random()` function in action:

```edgeql
select <int16>round(random() * 5);
```

So now we'll use it to update our `Person` objects (except for Jonathan, Dracula and Renfield) and give them all a random strength.

```edgeql
update Person
  filter .name not in {'Jonathan Harker', 'Count Dracula', 'Renfield'}
  set {
    strength := <int16>round(random() * 5)
  };
```

Now let's `select Person.strength;` and see if it works. The output should have mostly random numbers now:

```
{1, 1, 3, 2, 0, 5, 3, 0, 0, 5, 10, 2, 4, 0, 2, 5, 3, 4, 0, 1, 20, 5, 5, 5, 5, 5}
```

Looks like it worked.

So now let's overload the `fight()` function. Right now it only works for one `Person` vs. another `Person`, but in the book all the characters get together to try to defeat Dracula. We'll need to overload the function so that more than one character can work together to fight.

(Note: the array_join function in the signature joins an array of strings into a single string)

```sdl
function fight(people_names: array<str>, opponent: Person) -> str
  using (
    with
        people := (select Person filter contains(people_names, .name)),
    select
        array_join(people_names, ', ') ++ ' win!'
        if sum(people.strength) > (opponent.strength ?? 0)
        else (opponent.name ?? 'Opponent') ++ ' wins!'
  );
```

With this overload, we accept two arguments: an array of the names of the fighters and a `Person` object that is their opponent. You're seeing a couple of new standard library functions in use here that we'll dig into more later. For now, here are the basics you need to know:

- We call `contains`, passing our array of names and the `.name` property. This allows us to filter only `Person` objects with a `.name` that matches one of those in the array.
- `sum` in the next line of our `select` takes all the values in a set and adds them together. That gives us a total strength of the fighters to compare against their opponent's strength.

You may wonder why we provide a default value for the opponent name (`opponent.name ?? 'Opponent'`) but not for the names of the people in the group. This is because, if the array of names is empty, the group cannot win since no `Person` object will be returned from the query. We can't have a case where `people_names` is empty but the party won the fight, so no need for a fallback name to call the group! The `.name` property isn't required though, so since you pass in the opponent's `Person` object directly, you could pass in a powerful opponent without a name. The group is selected by their names, so there's no way to do the same for their side.

Note that overloading only works if the function signature is different. Here are the two signatures we have now for comparison:

```sdl
fight(one: Person, two: Person) -> str
fight(people_names: array<str>, opponent: Person) -> str
```

If we tried to overload our function with an input of `(Person, Person)`, it wouldn't work because that's the same as the original signature. EdgeDB uses the input we give it to know which form of the function to use, so without different signatures, it would have no way to decide between the two.

The function name is the same, but to call it, we enter an array of the fighters' names and the `Person` they are fighting.

Now Jonathan and Renfield are going to try to fight Dracula together. Good luck!

```edgeql
with
  party := ['Jonathan Harker', 'Renfield'],
  dracula := (select Person filter .name = 'Count Dracula'),
select fight(party, dracula);
```

So did they...

```
{'Count Dracula wins!'}
```

No, they didn't win. How about four people?

```edgeql
with
  party := ['Jonathan Harker', 'Renfield', 'Arthur Holmwood', 'The innkeeper'],
  dracula := (select Person filter .name = 'Count Dracula'),
select fight(party , dracula);
```

Much better:

```
{'Renfield, The innkeeper, Arthur Holmwood, Jonathan Harker win!'}
```

That's how function overloading works - you can create functions with the same name as long as the signature is different.

You see overloading in a lot of existing functions, such as {eql:func}`docs:std::sum` which we used earlier to get our party strength. {eql:func}`docs:std::to_datetime` has even more interesting overloading with all sorts of inputs to create a `datetime`.

`fight()` was pretty fun to make, but that sort of function is better done on the gaming side. So let's make a function that we might actually use. Since EdgeQL is a query language, the most useful functions are usually ones that make queries shorter.

Here is a simple one that tells us if a `Person` type has visited a `Place` or not:

```sdl
function visited(person: str, city: str) -> bool
  using (
    with person := (select Person filter .name = person),
    select city in person.places_visited.name
  );
```

Now our queries are much shorter:

```edgeql-repl
edgedb> select visited('Mina Murray', 'London');
{true}
edgedb> select visited('Mina Murray', 'Bistritz');
{false}
```

Thanks to the function, even more complicated queries are still quite readable:

```edgeql
select (
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' ++ <str>visited('Jonathan Harker', 'Romania')
);
```

This prints `{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`.

## More about Cartesian products and the coalescing operator

Now let's learn more about Cartesian products in EdgeDB. You might recall from the previous chapter that even a single `{}` input always results in an output of `{}`. That's why we had to change our `fight()` function to use the coalescing operator in the previous chapter. Let's dig a little deeper into why that is.

Remember, a `{}` has a length of 0 and anything multiplied by 0 is also 0. For example, let's try to add the names of places that start with b and those that start with x.

```edgeql
with b_places := (select Place filter Place.name ilike 'b%'),
     x_places := (select Place filter Place.name ilike 'x%'),
select b_places.name ++ ' ' ++ x_places.name;
```

The result may not be what you'd expect.

```
{}
```

Huh? It's an empty set! But a search for places that start with "b" gives us `{'Buda-Pesth', 'Bistritz'}`. Let's see if the same works when we concatenate with `++` as well.

```edgeql
select {'Buda-Pesth', 'Bistritz'} ++ {};
```

So that should give a `{}`. The output is...

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ select {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast or a conversion function.
```

Another surprise! This is an important point though: EdgeDB requires a cast for an empty set, because it won't try to guess at what type it is. There's no way to guess the type of an empty set if all we give it is `{}`, so EdgeDB won't try. You can probably guess that the same is true for array constructors too, so `select [];` returns an error: `QueryError: expression returns value of indeterminate type`.

Okay, one more time, this time making sure that the `{}` empty set is of type `str`:

```edgeql-repl
edgedb> select {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
```

Good, so we have manually confirmed that using `{}` with another set always returns `{}`. But what if we want to:

- Concatenate the two strings if they exist, and
- Return what we have if one is an empty set?

In other words, how to add `{'Buda-Peth', 'Bistritz'}` to another set and return the original `{'Buda-Peth', 'Bistritz'}` if the second is empty?

To do that we can again use the {eql:op}`coalescing operator <docs:coalesce>`:

```edgeql
with b_places := (select Place filter .name ilike 'b%'),
     x_places := (select Place filter .name ilike 'x%'),
select b_places.name ++ ' ' ++ x_places.name
  if exists b_places.name and exists x_places.name
  else b_places.name ?? x_places.name;
```

This returns:

```
{'Buda-Pesth', 'Bistritz'}
```

That's better.

But now back to Cartesian products. Remember, when we add or concatenate sets we are working with _every item in each set_ separately. So if we change the query to search for places that start with b (Buda-Pesth and Bistritz) and m (Munich):

```edgeql
with b_places := (select Place filter .name ilike 'b%'),
     m_places := (select Place filter .name ilike 'm%'),
select b_places.name ++ ' ' ++ m_places.name
  if exists b_places.name and exists m_places.name
  else b_places.name ?? m_places.name;
```

Then we'll get this result:

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

instead of something like 'Buda-Pesth, Bistritz, Munich'.

One way to join the two together without thinking about Cartesian multiplication is to turn them into an array. The {eql:func}`docs:std::array_agg` function will do this: it 'aggregates' them.

```edgeql
with b_places := (select Place filter .name ilike 'b%'),
     m_places := (select Place filter .name ilike 'm%'),
select array_agg(b_places.name) ++
  array_agg(m_places.name);
```

Using this gives us an output of `{['Buda-Pesth', 'Bistritz', 'Munich']}`.

But if we just want to stick with a set, there is an even easier way: just `union` the sets.

```edgeql
with b_places := (select Place filter .name ilike 'b%'),
  m_places := (select Place filter .name ilike 'm%'),
  select b_places.name union m_places.name;
```

And that will give us an output of `{'Buda-Pesth', 'Bistritz', 'Munich'}`.

With this, we don't need to worry about getting `{}` if we choose a letter like x. Let's look at every place that contains k or e:

```edgeql
with has_k := (select Place filter .name ilike '%k%'),
     has_e := (select Place filter .name ilike '%e%'),
     has_either := has_k union has_e,
select has_either.name;
```

This gives us the result:

```
{'Slovakia', 'Buda-Pesth', 'Castle Dracula'}
```

Similarly, you can use `?=` instead of `=` and `?!=` instead of `!=` when doing comparisons if you think one side might be an empty set. So then you can write a query like this:

```edgeql
with cities1 := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     cities2 := <str>{}, # Don't forget to cast to <str>
select cities1 ?= cities2;
```

and get the output

```
{false, false, false}
```

instead of `{}` for the whole thing. Also, two empty sets are treated as equal if you use `?=`. So this query:

```edgeql
select Vampire.lover.name ?= Crewman.name;
```

will return `{true}`. (Because Dracula has no lover and the Crewmen have no names so both sides return empty sets of type `str`.)

[Here is all our code so far up to Chapter 12.](code.md)

<!-- quiz-start -->

## Time to practice

1. Consider these two functions. Will EdgeDB allow both of them to be defined at the same time?

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

2. How about these two functions? Will EdgeDB allow both of them to be defined at the same time?

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

3. Will `select {} ?? {3, 4} ?? {5, 6};` work?

4. Will `select <int64>{} ?? <int64>{} ?? {1, 2}` work?

5. Trying to make a single string of everyone's name with `select array_join(array_agg(Person.name));` isn't working. What's the problem?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _One of the men gives his blood to try to save Lucy. Will it be enough?_
