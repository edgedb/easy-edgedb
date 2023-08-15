---
tags: Constraints, Deleting
leadImage: illustration_03.jpg
---

# Chapter 3 - Jonathan goes to Castle Dracula

In this chapter we encounter a vampire for the first time - or rather, _the_ vampire. As you can see, Dracula seems human but is actually quite different:

> Jonathan Harker has just arrived at Castle Dracula after a ride in the carriage through the mountains. The ride was terrible. There was snow, strange blue fires, and wolves everywhere. It was **night** when he arrived. He meets with Count Dracula. Dracula picks up all of Jonathan's luggage **with one hand**, they go inside, and they talk all night. Dracula always leaves **just before the sun rises** though. Curious! Days go by, and Jonathan still doesn't know that Dracula is a vampire. But he does notice something strange: the castle seems completely empty. If Dracula is so rich, where are his servants? Why is he so strong? Who is making his meals that he finds on the table? But Jonathan finds Dracula's stories of history very interesting, and so far is enjoying his trip.

Now we are completely inside Dracula's castle, so this is a good time to create a `Vampire` type. We can extend it from `abstract type Person` because that type has `name` and `places_visited`, which are good for `Vampire` too.

One possibility is adding `age` to `Person` so that all the other types can use it too. Then `Person' would look like this:

```sdl
abstract type Person {
  required name: str;
  multi places_visited: City;
  age: int16;
}
```

`int16` means a 16 bit (2 byte) integer, which has enough space for -32768 to +32767. That's enough for age, so we don't need the bigger `int32` or `int64` types which are much larger. We also don't want it to be a `required` property, because we don't care about everybody's age.

But we don't want `PC`s and `NPC`s to live up to 32767 years, so let's remove `age` from the abstract `Person` type and give it only to `Vampire` for now. We will think about the other types later. We'll make `Vampire` a type that extends `Person`, and adds age:

```sdl
type Vampire extending Person {
  age: int16;
}
```

Now we can create Count Dracula. We know that he lives in Romania, but Romania isn't a city. This is a good time to change the `City` type. We'll change the name to `Place` and make it an `abstract type`, and then `City` can extend from it. We'll also add a `Country` type that does the same thing. Now they look like this:

```sdl
abstract type Place {
  required name: str;
  modern_name: str;
  important_places: array<str>;
}

type City extending Place;

type Country extending Place;
```

We will also need to change `places_visited` in `Person` from `City` to `Place` so that it can include many things: the `City` types London and Bistritz, the `Country` type Hungary, and any other types we decide to add later that will extend `Place`.

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Place;
}
```

With these changes done, let's do a migration!

It's easy to make a `Country`, because its only required property is `name`. We'll quickly insert two `Country` objects for Hungary and Romania:

```edgeql
insert Country { name := 'Hungary' };
insert Country { name := 'Romania' };
```

## Capturing a select expression

With these countries added, we are now ready to insert Dracula.

We only know that Dracula has been in Romania, so his `places_visited` will be pretty easy: just select all the `Place` types and filter on `.name = 'Romania'`. When doing this, we put the `select` inside `()` brackets (parentheses). The parentheses are used to capture the result of the `select` query. In other words, the parentheses delimit (set the boundaries for) the `select` query which is then assigned to `places_visited`.

```edgeql
insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
  # .places_visited is the result of this select query.
};
```

The insert works, with a result that looks something like this: `{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`.

Let's check if `places_visited` worked. We only have one `Vampire` object now, so let's `select` it:

```edgeql
select Vampire {
  places_visited: {
    name
  }
};
```

This query gives us the following:

```
{default::Vampire {places_visited: {default::Country {name: 'Romania'}}}}
```

Perfect!

## Adding constraints

Now let's think about `age` again. It was easy to add `age` to the `Vampire` type, because they can live forever. But now we want to give `age` to `PC` and `NPC` too, who are humans who don't live forever (we don't want them living up to 32767 years). For this we can add a "constraint" (a limit). Instead of `age`, we'll give them a new scalar type called `HumanAge`. Then we can write `constraint` on it and use {ref}`one of the functions <docs:ref_datamodel_constraints>` that it can take. We will use the one called  `max_value()`.

Here's the signature for `max_value()`:

`std::max_value(max: anytype)`

The `anytype` part of the signature is interesting, because that means that the constraint can work on other types like strings too. With a constraint `max_value('B')` for example you couldn't use 'C', 'D', etc.

Now let's go back to our constraint for `HumanAge`, which is 120. The `HumanAge` type looks like this:

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

Remember, it's a scalar type because it can only have one value. Then we'll add it to the `NPC` type.

```sdl
type NPC extending Person {
  age: HumanAge;
}
```

This `HumanAge` scalar is our own type with its own name, but underneath it's an `int16` that can't be greater than 120. So let's do a migration now and try to add an `NPC` that is 130 years old. This `insert` should not work:

```edgeql
insert NPC {
  name := 'The innkeeper',
  age := 130
};
```

As expected, we are not allowed and an error shows up. Perfect.

```
edgedb error: ConstraintViolationError: Maximum allowed value for HumanAge is 120.
 Detail: Maximum allowed value for value of scalar type 'default::HumanAge' is 120.
```

Now let's insert the same innkeeper but give him an `age` of 30. This will now work: `{default::NPC {id: e7c4dd96-f2d4-11ed-a0c7-17fe2be02578}}`. Our `NPC` objects are now guaranteed to be no more than 120 years old.

## Deleting objects

Deleting in EdgeDB is very easy: just use the `delete` keyword. The `delete` keyword is similar to `select` in that it will apply to all types unless you `filter` them. This similarity to `select` might make you nervous, because if you type something like `select City;` then it will select all of them. Using `delete` is the same: `delete City;` deletes every object for the `City` type. That's why you should think carefully before deleting anything.

However, sometimes you may be prevented from deleting an object. Remember our two `Country` objects for Hungary and Romania? Let's try deleting them all. Interestingly, it won't work:

```edgeql
delete Country;
```

We got an error telling us that deleting a `Country` is not possible because it is still referenced by `places_visited` of a `Person`.

```
edgedb error: ConstraintViolationError: deletion of default::Country
(e9e8acda-f2d2-11ed-86e2-4bed5b30457e) is prohibited by link target policy
Detail: Object is still referenced in link places_visited of 
default::Person (ce8146ea-f2d3-11ed-92a8-237b79022c26).
```

That's Count Dracula who visited Romania getting in the way. Let's delete him first then:

```edgeql
delete Vampire;
```

Just like `insert`, using `delete` gives us the id numbers of the objects that are now deleted: `{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`.

Now we can try deleting all of the `Country` objects. Count Dracula is no longer in our way so the query will work:

```edgeql
delete Country;
```

We got confirmation that two `Country` objects have been deleted:

```
{
  default::Country {id: e988e476-f2d2-11ed-86e2-e34ef4a919b9},
  default::Country {id: e9e8acda-f2d2-11ed-86e2-4bed5b30457e},
}
```

Okay, let's insert both countries again so that we can delete them once more. This time we will try deleting them with a filter. First let's try deleting every `Country` object that has a name with "States" somewhere inside it. `ilike` will let us do that:

```edgeql
delete Country filter .name ilike '%States%';
```

Nothing matches, so the output is just an empty set of `{}` - we deleted nothing. Let's try again, deleting any `Country` object that has "ania" in its name:

```edgeql
delete Country filter .name ilike '%ania%';
```

We got a `{default::Country {id: 7f3c611c-ff43-11eb-af59-dfe5a152a5cb}}`, which is certainly Romania. Only Hungary is left. What if we want to see what we deleted? No problem - just put the `delete` inside parentheses and `select` it. Since EdgeDB just returns the objects that have been worked on, we can give them a shape to view them even as we are deleting them! So the query will look like this:

```edgeql
select (delete Country) {
  name
};
```

These queries are easier to read if you think about what expressions inside parentheses evaluate into. What does `delete Country` evaluate into? A set of `Country` objects. So the query is essentially just saying "select the country objects (that we are deleting) and display their name".

The output is `{default::Country {name: 'Hungary'}}`, showing us that we deleted Hungary. And now if we do `select Country` we get a `{}`, which confirms that we did delete them all.

Fun fact: `delete` statements in EdgeDB are actually {ref}`syntactic sugar <docs:ref_eql_statements_delete>` for `delete (select ...)`. You'll be learning something called `limit` in the next chapter with `select` and as you do so, keep in mind that you can apply the same to `delete` too.

We should probably delete that `City` object with `''` as its name that we inserted in the last chapter as we practiced indexing. That's easy:

```edgeql
delete City filter .name = '';
```

Finally, let's insert Hungary and Romania again to finish the section on deleting. Plus Count Dracula! With those three objects inserted again, we'll now leave them alone.

## The splat operator

Sometimes a query can take some time to type. Let's say we want to look up all of our `PC` objects and their properties, plus check whether their name has changed since Bram Stoker's book Dracula was published. Such a query would look like this:

```edgeql
select City {
  name,
  modern_name,
  important_places,
  id,
  name_has_changed := exists .modern_name
};
```

This gives us the following output:

```
{
  default::City {
    name: 'Munich',
    modern_name: {},
    important_places: {},
    id: 8a6d06bc-0b04-11ee-bd1f-67f7c83004cb,
    name_has_changed: false,
  },
  default::City {
    name: 'Buda-Pesth',
    modern_name: 'Budapest',
    important_places: {},
    id: 8a9e9222-0b04-11ee-bd1f-cfa57ad4cff6,
    name_has_changed: true,
  },
  default::City {
    name: 'Bistritz',
    modern_name: 'Bistrița',
    important_places: ['Golden Krone Hotel'],
    id: 8ac23b46-0b04-11ee-bd1f-cf07e2483e97,
    name_has_changed: true,
  },
}
```

Not bad! But we had to type quite a bit to select every property inside `City`. Wouldn't it be nice if EdgeDB had an operator that could do that for us?

It just so happens that EdgeDB since 2023 does have such an operator! The operator is called the splat operator because it uses a `*` which...looks like a splat. In other languages you sometimes see this called the 'global operator' which also has to do with importing or using everything in a namespace, so the `*` operator was well chosen. In SQL this is callect "select star", and the splat operator in EdgeDB is a better and more powerful version of that.

Let's try using the splat operator with `City` now.

```edgeql
select City {*};
```

And that's all there is to it! Here's the output:

```
{
  default::City {
    name: 'Munich',
    modern_name: {},
    important_places: {},
    id: 8a6d06bc-0b04-11ee-bd1f-67f7c83004cb,
  },
  default::City {
    name: 'Buda-Pesth',
    modern_name: 'Budapest',
    important_places: {},
    id: 8a9e9222-0b04-11ee-bd1f-cfa57ad4cff6,
  },
  default::City {
    name: 'Bistritz',
    modern_name: 'Bistrița',
    important_places: ['Golden Krone Hotel'],
    id: 8ac23b46-0b04-11ee-bd1f-cf07e2483e97,
  },
}
```

And if we want to include the computed `name_has_changed` property, we can just add it after the splat operator. So the quick query below will return the same output as the first query that needed all the extra typing.

```edgeql
select City {
  *,
  name_has_changed := exists .modern_name
};
```

If the splat operator is used for a type that is extended by other types, it will choose all of the properties that they have in common. Let's demonstrate that with three queries.

```
db> select PC{*};
{default::PC {name: 'Emil Sinclair', id: 8b0633d2-0b04-11ee-bd1f-2f48cb19fb35, class: Mystic}}

db> select NPC {*};
{
  default::NPC {name: 'Jonathan Harker', id: 8ae5923a-0b04-11ee-bd1f-e3545fe0bccf, age: {}},
  default::NPC {name: 'The innkeeper', id: 8bd08f10-0b04-11ee-bd1f-bf2be2316e84, age: 30},
}

db> select Person {*};
{
  default::PC {id: 8b0633d2-0b04-11ee-bd1f-2f48cb19fb35, name: 'Emil Sinclair'},
  default::Vampire {id: 8b4aefcc-0b04-11ee-bd1f-c36d2df0b47a, name: 'Count Dracula'},
  default::NPC {id: 8ae5923a-0b04-11ee-bd1f-e3545fe0bccf, name: 'Jonathan Harker'},
  default::NPC {id: 8bd08f10-0b04-11ee-bd1f-bf2be2316e84, name: 'The innkeeper'},
}
```

Notice the difference? The `PC` type contains a `class` that the two others don't, while `NPC` has an `age` property. The base `Person` type doesn't hold either of these two properties.

## The double splat operator

But wait, there's more! EdgeDB also has a double splat operator: `**` instead of `*`. Let's see what it does with `City`.

```edgeql
select City {**};
```

If you try that query...you'll get the same output. That's because `City` isn't linked to anything, and the double splat operator is used for showing all properties _and_ all links.

Let's demonstrate by querying all of our `Person` objects. We tried a query just before with `select Person {*};` which showed us all of the `Person` types, their `id` and their `name`. Could the double splat operator be any different? Let's give it a try.

```edgeql
select Person {**};
```

The output is...verbose!

```
{
  default::PC {
    id: 8b0633d2-0b04-11ee-bd1f-2f48cb19fb35,
    name: 'Emil Sinclair',
    places_visited: {},
  },
  default::Vampire {
    id: 8b4aefcc-0b04-11ee-bd1f-c36d2df0b47a,
    name: 'Count Dracula',
    places_visited: {
      default::Country {
        id: 8b2a64f0-0b04-11ee-bd1f-632e60d595d2,
        important_places: {},
        modern_name: {},
        name: 'Romania',
      },
    },
  },
  default::NPC {
    id: 8ae5923a-0b04-11ee-bd1f-e3545fe0bccf,
    name: 'Jonathan Harker',
    places_visited: {
      default::City {
        id: 8a6d06bc-0b04-11ee-bd1f-67f7c83004cb,
        important_places: {},
        modern_name: {},
        name: 'Munich',
      },
      default::City {
        id: 8a9e9222-0b04-11ee-bd1f-cfa57ad4cff6,
        important_places: {},
        modern_name: 'Budapest',
        name: 'Buda-Pesth',
      },
      default::City {
        id: 8ac23b46-0b04-11ee-bd1f-cf07e2483e97,
        important_places: ['Golden Krone Hotel'],
        modern_name: 'Bistrița',
        name: 'Bistritz',
      },
    },
  },
  default::NPC {
    id: 8bd08f10-0b04-11ee-bd1f-bf2be2316e84,
    name: 'The innkeeper',
    places_visited: {},
  },
}
```

This is because `Person` is linked to `Place` via the `places_visited` link, and the double splat operator shows you both an object's properties and the properties of the objects it links to. This operator only goes down to a depth of one, meaning that it won't follow the links of a linked object. This makes sense because in some databases you could see links that go on almost forever.

## Deleting Bistritz

Now that we know how to `delete`, let's try to get rid of our duplicate Bistritz. Later on we will learn how to do this smoothly thanks to learning how to `update` in Chapter 6, and how to manage deletion policies in Chapter 13. But even at this point we still know enough to make the deletion happen with a few extra steps.

Let's first do a query to remember what the two objects look like at the moment:

```edgeql
select City {*} filter .name = 'Bistritz';
```

And the output:

```
{
  default::City {
    name: 'Bistritz',
    modern_name: 'Bistrița',
    id: 8ac23b46-0b04-11ee-bd1f-cf07e2483e97,
    important_places: ['Golden Krone Hotel'],
  },
  default::City {
    name: 'Bistritz',
    modern_name: 'Bistrița',
    id: d1e38192-0bd6-11ee-ba45-d7ecb20723cf,
    important_places: {},
  },
}
```

Ah yes, we have a `City` called Bistritz that was inserted before we added the `important_places` property. There are two differences between the two: they have different `id` numbers, and one has an `important_places` property. Our plan is to delete one of them by filtering on its `id` property, so make a note of the `id` of the `City` that doesn't have any `important_places`. The `id` will naturally be different on your computer as an `id` is always unique.

Now if we wanted to delete both of them, we would simply type `delete City;`. However, this wouldn't work in any case. Give it a try and see what happens!

```edgeql
delete City;
```

The problem here is that Jonathan Harker is standing in our way. He is an `NPC` object with a link to the other `City` objects, and by default you can't delete an object that is being linked to by another one.

```
edgedb error: ConstraintViolationError: deletion of default::City (f801a034-387c-11ee-95af-87dfbf43e85c) is prohibited by link target policy
  Detail: Object is still referenced in link places_visited of default::Person (fc6522d6-387c-11ee-95af-f750f3ca3b62).
```

Within the limitations of what we know at the moment, the only thing we can do is to temporarily delete Jonathan Harker. We'll `insert` him again shortly, but in the meantime let's just get him out of the way:

```edgeql
delete NPC filter .name = 'Jonathan Harker';
```

And now that Jonathan is no longer standing in the way, let's try to delete one of the duplicate cities.

This query almost works but not quite:

```edgeql
delete City filter .id = 'd1e38192-0bd6-11ee-ba45-d7ecb20723cf';
```

Close! Remember the casting we did from a `str` to `uuid` in Chapter 2?

```
error: InvalidTypeError: operator '=' cannot be applied to operands of type 
'std::uuid' and 'std::str'
  ┌─ <query>:1:13
  │
1 │ delete City filter .id = 'd1e38192-0bd6-11ee-ba45-d7ecb20723cf';
  │             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 
  Consider using an explicit type cast or a conversion function.
```

So let's just cast the `str` to a `uuid` and it will now work:

```edgeql
delete City filter .id = <uuid>'d1e38192-0bd6-11ee-ba45-d7ecb20723cf';
```

And now it's gone! We are back to a single `City` object named Bistritz instead of two.

Finally, let's put Jonathan Harker back into the database. Soon we'll learn enough about EdgeDB so that we won't have to delete him like we did in this chapter.

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

[Here is all our code so far up to Chapter 3.](code.md)

<!-- quiz-start -->

## Time to practice

1. This query is trying to display every `NPC` along with the `name` plus every `City` type for each `NPC`, but it's giving an error. What is it missing?

   ```edgeql
   select NPC {
     name,
     cities := select City.name
   };
   ```

2. If the `City` type needed a required property called `population`, what would it look like? What type would 'population' be?

3. This query wants to display `name` twice for some reason but is giving an error. Can you think of a way to do it?

   ```edgeql
   select Person {
     name,
     name
   };
   ```

   (Hint: the problem is that the name `name` is being used twice)

4. People keep trying to make characters with negative ages. Can you think of a constraint that can stop this?

   Hint: the current constraint is `max_value(120);`

5. Can you insert a HumanAge type?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan: "This Count Dracula knows so much about history! I'm glad I came."_
