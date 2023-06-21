---
tags: Complex Inserts, Schema Cleanup, Triggers
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
  overloaded required major: str {
    default := 'pound'
  }
  overloaded required minor: str {
    default := 'shilling'
  }
  overloaded required minor_conversion: int64 {
    default := 20
  }
  overloaded sub_minor: str {
    default := 'pence'
  }
  overloaded sub_minor_conversion: int64 {
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

## Triggers

Let's give some thought to the actual users of our game - the people who will be signing up to make player characters to try to save the world from (or to help?) Count Dracula.

However the game takes shape, it will require some sort of account type to hold the information for the people using our game. A simplified example of what we might need is as follows:

```sdl
type Account {
  required name: str;
  required address: str;
  required username: str;
  required credit_card: CreditCardInfo {
    on source delete delete target;
  }
  multi pcs: PC;
}

type CreditCardInfo {
  required name: str;
  required number: str;

  link card_holder := .<credit_card[is Account]
}
```

Every single property in these types is required, which makes sense - it's all personal information required to create accounts and pay for them.

Now let's think about what happens when an `Account` gets deleted. In almost every country, a company is legally bound to remove a user's personal information when they ask for an account to be deleted. With the `on source delete delete target` deletion policy, we have the `credit_card` link set up to delete any linked `CreditCardInfo` when an `Account` object is deleted. (A reminder: `on source delete delete target` means that when the source of the link is deleted, the target of the link gets deleted as well.) Deleting an `Account` object will delete absolutely everything to do with the user of our game.

However, users often delete their accounts but then want to restore them again! But we can't keep the `Account` and `CreditCardInfo` objects around just in case, because we are legally obliged to remove the user's information. An easy way to solve this problem is by using _triggers_, which were added in EdgeDB 3.0. A trigger represents some sort of action that we want to take place every time an object is inserted, updated, or deleted. Let's first take a look at the official syntax for triggers to get an idea of how to use them:

```
type type-name "{"
  trigger name
  after
    {insert | update | delete} [, ...]
    for {each | all}
    do expr
"}"
```

In other words:

- Decide on a name for a trigger (like `validate_input` or `check_length`),
- Add the keyword `after`,
- Decide for which cases we want a trigger to happen (only on `insert`, or for `insert` and `update`, etc.),
- Decide whether a trigger should happen on `each` object or once per query with `all`,
- And finally the expression of the trigger itself.

Inside a trigger we get access to the old object using `__old__` and the new object using `__new__`, depending on the operation. When you `delete` there is no `__new__` object to access, nor is there an `__old__` object to access when doing an `insert`.

And now back to our `Account` type. To keep a minimal amount of information after a user's account is deleted, we can create a new type that holds this information. We can call this type `MinimalUserInfo`, because it will only hold their username and the `PC` objects they made. (Perhaps we will hold on to `PC` objects for 30 days or so in case a user changes their mind)

The `MinimalUserInfo` type is pretty simple:

```sdl
type MinimalUserInfo {
  username: str;
  multi pcs: PC;
}
```

And now let's add the trigger to `Account` that inserts a `MinimalUserInfo` object every time an `Account` object is deleted. With this trigger in place, we can freely delete any `Account` objects and the user's personal information will be removed: name, address, and credit card info. But a `MinimalUserInfo` will always be created at the same time which holds the `username` and the `PC` objects linked to it.

All together, the changes now look as follows:

```sdl
type Account {
  required name: str;
  required address: str;
  required username: str;
  required credit_card: CreditCardInfo {
    on source delete delete target;
  }
  multi pcs: PC;

  trigger user_info_insert after delete for each do (
  insert MinimalUserInfo {
    username := __old__.name,
    pcs := __old__.pcs
  }
);
}

type CreditCardInfo {
  required name: str;
  required number: str;

  link card_holder := .<credit_card[is Account]
}

type MinimalUserInfo {
  username: str;
  multi pcs: PC;
}
```

Okay, let's do a migration and then insert an `Account` object!

```
insert Account {
  name := 'Deborah Brown',
  address := '10 Main Street',
  username := 'deb_deb_999',
  credit_card := (insert CreditCardInfo 
    {  name := 'DEBORAH LAURA BROWN',
       number := '000-000-000' }
  ),
  pcs := (insert PC 
    {  name := 'LordOfSalty',
       class := Class.Rogue
    }
  )
};
```

We have only one `Account` object so far so let's query with `select Account {**};` to see all of the information available. Here is the output:

```
{
  default::Account {
    id: 6f67992a-0d90-11ee-b3bb-8f877d23b651,
    name: 'Deborah Brown',
    address: '10 Main Street',
    username: 'deb_deb_999',
    pcs: {
      default::PC {
        last_appearance: {},
        first_appearance: {},
        degrees: {},
        title: {},
        age: {},
        is_single: true,
        strength: {},
        name: 'LordOfSalty',
        pen_name: 'LordOfSalty',
        conversational_name: 'LordOfSalty',
        id: 6f67f154-0d90-11ee-b3bb-37992cdeda59,
        class: Rogue,
        created_at: <datetime>'2023-06-18T04:27:29.059881Z',
        number: 5,
      },
    },
    credit_card: default::CreditCardInfo {
      id: 6f678106-0d90-11ee-b3bb-13592cce3a16,
      name: 'DEBORAH LAURA BROWN',
      number: '000-000-000',
    },
  },
}
```

Looks good! That is, until Deborah decides she has been playing too many vampire games recently and would like to delete her account. We are sad to see her go but comply with her request:

```edgeql
delete Account filter .name = 'Deborah Brown';
```

And now Deborah's personal information is all gone, as requested. But thanks to the trigger we added, we now have a `MinimalUserInfo` object in the database that was added automatically at the moment that we deleted Deborah's account. Let's take a look at it:

```
select MinimalUserInfo {**};
```

Here is the output:

```
{
  default::MinimalUserInfo {
    id: 5c17b0de-0d91-11ee-946c-6f411083ab25,
    username: 'deb_deb_999',
    pcs: {
      default::PC {
        last_appearance: {},
        first_appearance: {},
        degrees: {},
        title: {},
        age: {},
        is_single: true,
        strength: {},
        name: 'LordOfSalty',
        pen_name: 'LordOfSalty',
        conversational_name: 'LordOfSalty',
        id: 59cbd4b8-0d91-11ee-946c-a3693fe5d218,
        class: Rogue,
        created_at: <datetime>'2023-06-18T04:34:02.301357Z',
        number: 8,
      },
    },
  },
}
```

With that we are holding none of Deborah's personal information anymore. All we know is that some user called `deb_deb_999` had a `PC` called `LordOfSalty` - no personal info anywhere! And if Deborah decides that she wants to get back into the world of vampire gaming she can choose to restore her account - as long as she remembers her username.

[Here is all our code so far up to Chapter 18.](code.md)

<!-- quiz-start -->

## Time to practice

1. The Goldmark was used in Germany during the time of Bram Stoker's Dracula. One Goldmark had 100 Pfennig. How would you make this type?

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
