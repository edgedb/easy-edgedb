---
tags: Overloading Functions, Coalescing
---

# Chapter 12 - From bad to worse

There is no good news for our heroes this chapter:

> Dracula continues to break into Lucy's room every time people don't listen to Van Helsing's advice, and every time the men give their blood to save her. Dracula always turns into a cloud to sneak in, drinks her blood and sneaks away before morning. Lucy is getting so weak that it's a surprise that she's still alive.
>
> Meanwhile, Renfield breaks out of his cell and attacks Dr. Seward with a knife, causing him to bleed. The moment Renfield sees Dr. Seward's blood he stops and tries to drink it, repeating: “The blood is the life! The blood is the life!”. The asylum security men take Renfield away and Dr. Seward is left confused and trying to understand him. Dr. Seward thinks there is a connection between him and the other events.
>
> That night, a wolf controlled by Dracula breaks the windows of Lucy's room and Dracula is able to get in again...

But there is good news for us, because we are going to keep learning about Cartesian products, plus how to overload a function.

## Overloading functions

Functions can be overloaded in EdgeDB. Overloading a function means to create a function with the same name as another function but with a different signature, thereby allowing it to take in and return different types. The `cal::to_local_date()` function that we saw in Chapter 9 is an example of an overloaded function as there are three ways to use it:

```
cal::to_local_date(s: str, fmt: optional str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

Last chapter, we gave every character a strength of 5 and used the `fight()` function for a few of them. That's why the Innkeeper defeated Dracula, which is obviously not what would really happen. We should give Dracula a more realistic strength, and randomize some of the strength values for some of our characters.

Jonathan Harker is just a human but is still quite strong. We'll let him keep his strength of 5. We'll treat that as the maximum strength for a human, except Renfield who is a bit unique - we gave him a strength of 10 when we inserted him. And we'll make sure Count Dracula gets 20 strength, because he's Dracula. So we only have to update Count Dracula's strength:

```edgeql
update Vampire
filter .name = 'Count Dracula'
set {
  strength := 20
};
```

Every other human should have a strength between 0 and 5. EdgeDB has a function called {eql:func}`docs:std::random` that gives a `float64` in between 0.0 and 1.0. If we multiply this by 5, we will get a `float64` in between 0.0 and 5.0. There is another function called {eql:func}`docs:std::round` that rounds numbers, so we'll use that too, and finally cast it to an `<int16>`. Give this a try a few times to see the `random()` function in action:

```edgeql
with before_rounded := random() * 5,
  after_rounded := round(before_rounded),
  select (before_rounded, after_rounded);
```

The number will differ every time, but the output will look something like `{(3.9557656033819555, 4)}`. After that we'll just need to cast the rounded number into an `int16` and that will work as a random `strength` value for each of our `Person` objects (except for Jonathan, Dracula and Renfield). Here is what the update looks like:

```edgeql
update Person
  filter .name not in {'Jonathan Harker', 'Count Dracula', 'Renfield'}
  set {
    strength := <int16>round(random() * 5)
  };
```

The vampire women are also pretty strong. Let's give them a random strength plus 5. This will ensure that they are stronger than average humans but not as strong as Dracula:

```edgeql
update MinorVampire
  set {
    strength := <int16>round(random() * 5) + 5
  };
```

Now let's `select Person { name, strength };` and see if it works. The output should have mostly random numbers now, something similar to this:

```
{
  default::Crewman {name: 'Crewman 1', strength: 1},
  default::Crewman {name: 'Crewman 2', strength: 4},
  default::Crewman {name: 'Crewman 3', strength: 3},
  default::Crewman {name: 'Crewman 4', strength: 2},
  default::Crewman {name: 'Crewman 5', strength: 3},
  default::MinorVampire {name: 'Vampire Woman 1', strength: 1},
  default::MinorVampire {name: 'Vampire Woman 2', strength: 4},
  default::MinorVampire {name: 'Vampire Woman 3', strength: 3},
  default::Sailor {name: 'The Captain', strength: 0},
  default::Sailor {name: 'Petrofsky', strength: 3},
  default::Sailor {name: 'The First Mate', strength: 4},
  default::Sailor {name: 'The Cook', strength: 1},
  default::PC {name: 'Emil Sinclair', strength: 5},
  default::PC {name: 'Max Demian', strength: 3},
  default::NPC {name: 'Renfield', strength: 10},
  default::NPC {name: 'Jonathan Harker', strength: 5},
  default::NPC {name: 'The innkeeper', strength: 2},
  default::NPC {name: 'Mina Murray', strength: 2},
  default::NPC {name: 'Quincey Morris', strength: 3},
  default::NPC {name: 'Arthur Holmwood', strength: 2},
  default::NPC {name: 'John Seward', strength: 1},
  default::NPC {name: 'Abraham Van Helsing', strength: 1},
  default::NPC {name: 'Lucy Westenra', strength: 3},
  default::Vampire {name: 'Count Dracula', strength: 20},
}
```

Looks like it worked.

So now let's overload the `fight()` function. Right now it only works for one `Person` vs. another `Person`, but in the book all the characters get together to try to defeat Dracula. We'll need to overload the function so that more than one character can work together to fight.

```sdl
function fight(people_names: array<str>, opponent: Person) -> str
  using (
    with
        people := (select Person filter contains(people_names, .name)),
    select
        array_join(people_names, ', ') ++ ' win!'
        if sum(people.strength) > (opponent.strength ?? 0)
        else opponent.name ++ ' wins!'
  );
```

With this overloaded function we accept two arguments: an array of the names of the fighters and a `Person` object that is their opponent. You're seeing a couple of new standard library functions in use here that we'll dig into more later. For now, here are the basics you need to know:

- We call `contains`, passing our array of names and the `.name` property. This allows us to filter only `Person` objects with a `.name` that matches one of those in the array.
- `sum` in the next line of our `select` takes all the values in a set and adds them together. That gives us a total strength of the fighters to compare against their opponent's strength.

Note that overloading only works if the function signature is different. Here are the two signatures we have now for comparison:

```sdl
fight(one: Person, two: Person) -> str
fight(people_names: array<str>, opponent: Person) -> str
```

If we tried to overload our function with an input of `(Person, Person)`, it wouldn't work because that's the same as the original signature. EdgeDB uses the input we give it to decide which form of the function to use, so without different signatures, it would have no way to decide between the two.

The function name is the same, but to call it, we enter an array of the fighters' names and the `Person` they are fighting.

Let's do a migration now and see what happens if Jonathan and Renfield try to fight Dracula together. The migration output is a little interesting, as EdgeDB simply asks us if we created a function `default::fight` - it doesn't ask us if we overloaded a function. For humans the function looks the same because of the function name (which is what makes overloading convenient), but as far as EdgeDB is concerned this simply an entirely new function.

```
c:\easy-edgedb>edgedb migration create
Connecting to an EdgeDB instance at localhost:10716...
did you create function 'default::fight'? [y,n,l,c,b,s,q,?]
> y
```

Now it's time to join together to fight Dracula. Good luck!

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

No, they didn't win. How about five people?

```edgeql
with
  party := ['Jonathan Harker', 'Renfield', 'Arthur Holmwood', 
    'The innkeeper', 'Lucy Westenra'],
  dracula := (select Person filter .name = 'Count Dracula'),
select fight(party , dracula);
```

At this point it is most likely that the random `strength` values for everyone together will be greater than 20 and they will finally win. Then we will see the following output:

```
{'Jonathan Harker, Renfield, Arthur Holmwood, The innkeeper, Lucy Westenra win!'}
```

So that's how function overloading works - you can create functions with the same name as long as the signature is different.

`fight()` was pretty fun to make, but that sort of function is better done on the gaming side. So let's make a function that we might actually use. Since EdgeQL is a query language, the most useful functions are usually ones that make queries shorter.

Here is a simple one that tells us if a `Person` type has visited a `Place` or not:

```sdl
function visited(person: str, city: str) -> bool
  using (
    with person := (select Person filter .name = person),
    select city in person.places_visited.name
  );
```

Pretty simple! Let's add it to the schema and do a migration. Now our queries are much shorter:

```
db> select visited('Mina Murray', 'London');
{true}
db> select visited('Mina Murray', 'Bistritz');
{false}
```

Thanks to the function, even more complicated queries are still quite readable:

```edgeql
select (
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' 
    ++ <str>visited('Jonathan Harker', 'Romania')
);
```

This prints `{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`.

## More about Cartesian products and the coalescing operator

Now let's learn more about Cartesian products in EdgeDB. You might recall from the previous chapter that even a single empty set always results in an output of `{}` when working with multiple sets:

```
edgedb> select 'My name is ' ++ <str>{} ++ '!';
{}
```

That's why we had to change our `fight()` function to use the coalescing operator in the previous chapter. Let's dig a little deeper into why that is.

Remember, a `{}` has a length of 0 and anything multiplied by 0 is also 0. For example, let's try to concatenate the names of places that start with b with those that start with x.

```edgeql
with b_places := (select Place filter Place.name ilike 'b%'),
     x_places := (select Place filter Place.name ilike 'x%'),
select b_places.name ++ ' ' ++ x_places.name;
```

Yep, it's an empty set.

```
{}
```

Let's explore this a bit more. A search for places that start with "b" should give us `{'Buda-Pesth', 'Bistritz'}`. Let's manually type out the city names this time just to make sure and add an empty set after:

```edgeql
select {'Buda-Pesth', 'Bistritz'} ++ {};
```

So that should give a `{}`. The output is...

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ select {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider 
using an explicit type cast or a conversion function.
```

Ah, that's right - we saw one example of an empty set with a cast in the last chapter when we tried the query `select <str>{} ?? 'Count Dracula is now in Whitby';`. EdgeDB requires a cast for an empty set, because there's no way to know the type of a set if all EdgeDB sees is `{}`.

You can probably guess that the same is true for array or set constructors too, so `select [];` returns an error: `QueryError: expression returns value of indeterminate type`. And `select {};` too.

Okay, one more time, this time making sure that the `{}` empty set is of type `str`:

```
db> select {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
```

Good, so we have manually confirmed that using `{}` with another set always returns `{}`. But we would like to do the following:

- Concatenate the two strings if they exist, and
- Return what we have if one is an empty set.

In other words, we would like to add `{'Buda-Pesth', 'Bistritz'}` to another set and return the original `{'Buda-Pesth', 'Bistritz'}` if the second is empty. So we use the {eql:op}`coalescing operator <docs:coalesce>` to make it work:

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

But now back to Cartesian products. Remember, when we add or concatenate sets we are working with _every item in each set_ separately. Let's see what happens if we change the query to search for places that start with a `b` (Buda-Pesth and Bistritz) and `m` (Munich). We would like to get the result `{'Buda-Pesth, Bistritz, Munich'}`, but it doesn't quite work that way:

```edgeql
with b_places := (select Place filter .name ilike 'b%'),
     m_places := (select Place filter .name ilike 'm%'),
select b_places.name ++ ' ' ++ m_places.name
  if exists b_places.name and exists m_places.name
  else b_places.name ?? m_places.name;
```

We get this result instead:

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

In other words, we concatenated each item inside each set with each item inside the other. How can we just join one set with another one instead?

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
{'Slovakia', 'France', 'Castle Dracula', 'Buda-Pesth'}
```

Similarly, you can use `?=` instead of `=` and `?!=` instead of `!=` when doing comparisons if you think one side might be an empty set. So if you can write a query like this:

```edgeql
with city_names := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     city_name := (select City.name filter City.name = "Hi I'm a City"),
select city_names = city_name;
```

It will return `{}` because `city_name` has returned an empty set. Now if we change `=` to `?=` then the query will work as we would like:

```edgeql
with city_names := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     city_name := (select City.name filter City.name = "Hi I'm a City"),
select city_names ?= city_name;
```

This will return the output `{false, false, false}` instead of `{}` for the whole thing. 

In addition, two empty sets are treated as equal if you use `?=`. So this query will return `{true}`:

```edgeql
select Vampire.lovers.name ?= Crewman.lovers.name;
```

It returns `{true}` because neither Dracula nor any of the Crewmen have a lover: both sides of the query return empty sets of type `str`. If we had used `=` instead of `?=` in this case, we would have just seen an empty set.

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

4. Will `select <int64>{} ?? <int64>{} ?? {1, 2};` work?

5. Trying to make a single string of everyone's name with `select array_join(array_agg(Person.name));` isn't working. What's the problem?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _One of the men gives his blood to try to save Lucy. Will it be enough?_
