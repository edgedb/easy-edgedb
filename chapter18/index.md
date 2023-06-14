---
tags: Complex Inserts, Schema Cleanup
---

# Chapter 18 - Using Dracula's own weapon against him

> Van Helsing was correct: Mina is connected to Dracula. Van Helsing continues to use hypnotism to find out more about where Dracula is and what he is doing. 
>
> Meanwhile, Jonathan does a lot of investigation into Dracula's activities in London. He visits all the companies that were involved in selling Dracula's house, and some moving companies who moved his coffins around. Jonathan is becoming more and more confident, and never stops working to find Dracula.
>
> Our heroes eventually find Dracula's other house in London with all his money. Knowing that he will come to get it, they wait for him to arrive...Suddenly, Dracula runs into the house and attacks. Jonathan hits out with his knife, and cuts Dracula's bag with all his money. Dracula grabs some of the money that fell and jumps out the window. He yells at them: "You shall be sorry yet, each one of you! You think you have left me without a place to rest; but I have more. My revenge is just begun!" Then he disappears.

This is a good reminder that we should probably think about money in our game. The characters have been to countries like England, Romania, and Germany, and each of those have their own money. An `abstract type` seems to be a good choice here: we should create an `abstract type Currency` that we can extend for all the other types of money.

Now, there is one difficulty: in the 1800s, monetary systems were more complicated than they are today. In England, for example it wasn't 100 pence to 1 pound, it was as follows:

- 12 pence (the smallest coin) made one shilling,
- 20 shillings made one pound, thus
- 240 pence per pound.

(There was also a _halfpenny_ that was half of one pence, but let's not get into that much detail in our game.)

To reflect this, we'll say that `Currency` has three properties: `major`, `minor`, and `sub_minor`. Each one of these will have an amount, and finally there will be a number for the conversion, plus an `owner: Person` link. So `Currency` will look like this:

```sdl
abstract type Currency {
  required owner: Person;

  required major: str;
  required major_amount: int64 {
    default := 0;
    constraint min_value(0);
  }

  minor: str;
  minor_amount: int64 {
    default := 0;
    constraint min_value(0);
  }
  minor_conversion: int64;

  sub_minor: str;
  sub_minor_amount: int64 {
    default := 0;
    constraint min_value(0);
  }
  sub_minor_conversion: int64;
}
```

You'll notice that only `major` properties are `required`, because some currencies don't even have things like cents. Two examples from modern times are the Japanese yen and Korean won that are just a single money unit and a number.

We also gave it a constraint of `min_value(0)` so that characters won't be able to buy with money they don't have. Money can be negative in real life, but we don't need to think about complicated subjects like credit and negative money in our game.

Then comes our first currency: the `Pound` type. The `minor` property is called `'shilling'`, and we use `minor_conversion` to get the amount in pounds. The same thing happens with `'pence'`. Then our characters can collect various coins but the final value can still quickly be turned into pounds. Here's the `Pound` type:

```sdl
type Pound extending Currency {
  overloaded required property major {
    default := 'pound'
  }
  overloaded required property minor {
    default := 'shilling'
  }
  overloaded required property minor_conversion {
    default := 20
  }
  overloaded property sub_minor {
    default := 'pence'
  }
  overloaded property sub_minor_conversion {
    default := 240
  }
}
```

Now let's do a schema migration and try it out. First we'll give Dracula some money. We'll give him 2500 pounds, 50 shillings, and 200 pence. Maybe that's a lot of money in 1893.

```edgeql
insert Pound {
  owner := (select Person filter .name = 'Count Dracula'),
  major_amount := 2500,
  minor_amount := 50,
  sub_minor_amount := 200
};
```

Then we can use the conversion rates to display the total amount he owns in pounds:

```edgeql
select Currency {
  owner: {name},
  total := .major_amount + (.minor_amount / .minor_conversion) 
    + (.sub_minor_amount / .sub_minor_conversion)
};
```

Based on the results of the query, he has about 2503 pounds:

```
{default::Pound {owner: default::Vampire {name: 'Count Dracula'}, total: 2503.3333333333335}}
```

We know that Arthur (now called Lord Godalming) has all the money he needs, but the others we aren't sure about. Let's give a few of them a random amount of money, and also `select` it at the same time to display the result. For the random number we'll use the method we used for `strength` before: `round()` on a `random()` number multiplied by the maximum.

Finally, when displaying the total we will cast it to a `decimal` type. With this, we can display the number of pounds as something like 555.76 instead of 555.76545256. For this we use the same `round()` function, but using the last signature:

```sdl
std::round(value: int64) -> float64
std::round(value: float64) -> float64
std::round(value: bigint) -> bigint
std::round(value: decimal) -> decimal
std::round(value: decimal, d: int64) -> decimal
```

That signature has an extra `d: int64` part for the number of decimal places we want to give it.

All together, it looks like this:

```edgeql
select (for character in {'Jonathan Harker', 'Mina Murray',
  'The innkeeper', 'Emil Sinclair'}
  union (
    insert Pound {
      owner := assert_single((select Person filter .name = character)),
      major_amount := <int64>round(random() * 500),
      minor_amount := <int64>round(random() * 100),
      sub_minor_amount := <int64>round(random() * 500)
  })) {
  owner: {
    name
  },
  pounds := .major_amount,
  shillings := .minor_amount,
  pence := .sub_minor_amount,
  total_pounds :=
    round(<decimal>(.major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)), 2)
};
```

And then it will give a result similar to this with our collections of money, each with an owner:

```
{
  default::Pound {
    owner: default::NPC {name: 'Jonathan Harker'},
    pounds: 386,
    shillings: 80,
    pence: 184,
    total_pounds: 390.77n,
  },
  default::Pound {
    owner: default::NPC {name: 'Mina Murray'},
    pounds: 385,
    shillings: 57,
    pence: 272,
    total_pounds: 388.98n,
  },
  default::Pound {
    owner: default::NPC {name: 'The innkeeper'},
    pounds: 374,
    shillings: 40,
    pence: 187,
    total_pounds: 376.78n,
  },
  default::Pound {
    owner: default::PC {name: 'Emil Sinclair'},
    pounds: 20,
    shillings: 86,
    pence: 1,
    total_pounds: 24.30n,
  },
}
```

(If you don't want to see the `n` for the `decimal` type, just cast it into a `<float32>` or `<float64>`.)

You'll notice now that there could be some debate on how to show money. Should it be a `Currency` that links to an owner? Or should it be a `Person` that links to a property called `money`? Our way might be easier for a realistic game, simply because there are many types of `Currency`. If we chose the other method, we would have one `Person` type linked to every type of currency, and most of them would be zero. But with our method, we only have to create 'piles' of money when a character starts owning them. Or these 'piles' could be things like purses and bags, and then we could change `required owner: Person;` to `optional owner: Person;` if it's possible for a character in the game to lose them.

Of course, if we only had one type of money then it would be simpler to just put it inside the `Person` type. We won't do this in our schema, but let's imagine how to do it. If the game were only inside the United States, it would be easier to just do this without an abstract `Currency` type:

```sdl
type Dollar {
  required dollars: int64;
  required cents: int64;
  property total_money := .dollars + (.cents / 100)
}
```

The `total_money` type, by the way, will become a `float64` because of the `/ 100` part. We can confirm this with a quick query:

```edgeql
select (100 + (55 / 100)) is float64;
```

The output: `{true}`.

We can see the same when we make an insert and use `select` to check the `total_money` property:

```edgeql
select (
  insert Dollar {
    dollars := 100,
    cents := 55
  }
) {
  total_money
};
```

Then the output would look like `{default::Dollar {total_money: 100.55}}`.

Dollars won't be used in this game, but if they were, we'd create it with `type Dollar extending Currency`.

One final note: the dollar's `total_money` property is just created by dividing by 100, so it's using `float64` in a limited fashion (which is good). But you want to be careful with floats because they are not always precise, and if we were to need to divide by 3 for example we would get results like `100 / 3 = 33.33333333`...not very good for actual currency. So in that case it would be better to stick to integers.

## Cleaning up the schema

We've come a long way since first setting up our database and doing our early object inserts. Let's look back and see how things might change if we were starting again, knowing what we now know.

First, we have two inserts here where we could only have one.

```edgeql
insert City {
  name := 'Munich',
};

insert City {
  name := 'London',
};
```

We'll change that to an insert with a `for` loop:

```edgeql
for city_name in {'Munich', 'London'}
union (
  insert City {
    name := city_name
  }
);
```

Then we'll do the same for the four `Country` types that we inserted (Hungary, Romania, France, Slovakia). Now they are a single insert:

```edgeql
for country_name in {'Hungary', 'Romania', 'France', 'Slovakia'}
union (
  insert Country {
    name := country_name
  }
);
```

The other `City` inserts are a bit different: some have `modern_name` and others have `population`. In a real game we would insert them all in this sort of form, all at once:

```edgeql
for city in {
    ('City 1\'s name', 'City 1\'s modern name', 800),
    ('City 2\'s name', 'City 2\'s modern name', 900),
    ('City 3\'s name', 'City 3\'s modern name', 455),
  }
union (
  insert City {
    name := city.0,
    modern_name := city.1,
    population := city.2
  }
);
```

And we would do the same with all the `NPC` types, their `first_appearance` data, and so on. But we don't have that many cities and characters to insert in this tutorial so we don't need to be so systematic yet.

We can also turn the inserts for the `Ship` type into a single one. Right now it looks like this:

```edgeql
for n in {1, 2, 3, 4, 5}
union (
  insert Crewman {
    number := n,
    first_appearance := cal::to_local_date(1893, 7, 6),
    last_appearance := cal::to_local_date(1893, 7, 16),
  }
);

insert Sailor {
  name := 'The Captain',
  rank := Rank.Captain
};

insert Sailor {
  name := 'The First Mate',
  rank := Rank.FirstMate
};

insert Sailor {
  name := 'The Second Mate',
  rank := Rank.SecondMate
};

insert Sailor {
  name := 'The Cook',
  rank := Rank.Cook
};

insert Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

Let's put that all together:

```edgeql
insert Ship {
  name := 'The Demeter',
  sailors := {
    (insert Sailor {
      name := 'The Captain',
      rank := Rank.Captain
    }),
    (insert Sailor {
      name := 'The First Mate',
      rank := Rank.FirstMate
    }),
    (insert Sailor {
      name := 'The Second Mate',
      rank := Rank.SecondMate
    }),
    (insert Sailor {
      name := 'The Cook',
      rank := Rank.Cook
    })
  },
  crew := (
    for n in {1, 2, 3, 4, 5}
    union (
      insert Crewman {
        number := n,
        first_appearance := cal::to_local_date(1893, 7, 6),
        last_appearance := cal::to_local_date(1893, 7, 16),
      }
    )
  )
};
```

Much better!

[Here is all our code so far up to Chapter 18.](code.md)

<!-- quiz-start -->

## Time to practice

1. During the time of Dracula, the Goldmark was used in Germany. One Goldmark had 100 Pfennig. How would you make this type?

2. Try adding two annotations to this type. One should be called `description` and mention that `One mark = 100 Pfennig`. The other should be called `note` and mention the types of coins there are.

   [Here are the types of coins](https://en.wikipedia.org/w/index.php?title=German_gold_mark&oldid=972733514#Base_metal_coins): 1, 2, 5, 10, 20, 25 Pfennig coins.

3. A vampire named Godbrand has just attacked a village and turned three villagers into `MinorVampire`s. How would you insert all four of them at once?

   Here is their data (name, date of birth (`first_appearance`), date turned into a MinorVampire (`last_appearance`)):

   ```
   ('Fritz Frosch', '1850-01-15', '1893-09-11'),
   ('Levanta Sinyeva', '1862-02-24', '1893-09-11'),
   ('김훈', '1860-09-09', '1893-09-11'),
   ```

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Only Mina can tell them where Dracula has gone._
