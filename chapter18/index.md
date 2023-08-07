---
tags: Complex Inserts, Schema Cleanup, Triggers
---

# Chapter 18 - Using Dracula's own weapon against him

> Van Helsing was correct: Mina is connected to Dracula. Van Helsing continues to use hypnotism to find out more about where Dracula is and what he is doing. 
>
> Meanwhile, Jonathan does a lot of investigation into Dracula's activities in London. He visits all the companies that were involved in selling Dracula's house, and some moving companies who moved his coffins around. Jonathan is becoming more and more confident, and only has one thought in his mind: find Dracula, destroy him, and save Mina.
>
> Our heroes eventually find Dracula's other house in London with all of his money. Knowing that Dracula will come to get it, they wait for him to arrive.
>
> Suddenly, Dracula runs into the house and attacks. Jonathan hits out with his knife, and cuts Dracula's bag with all his money. Dracula grabs some of the money that fell and jumps out the window. He yells at them: "You shall be sorry yet, each one of you! You think you have left me without a place to rest; but I have more. My revenge is just begun!" Then he disappears.

This is a good reminder that we should probably think about money in our game. Money in a regular fantasy game is generally pretty easy: you can use a parameter called `gold` or something, call that the currency and give everyone a `gold` property. But our game is based on the real world. The characters have been to countries like England, Romania, and Germany, each of which have their own money.

On top of this is an extra difficulty: in the 1800s, monetary systems were more complicated than they are today. For example, the United Kingdom at the time didn't use a simple 100 pence to 1 pound conversion. Instead, it was as follows:

- 12 pence (the smallest coin) made one shilling,
- 20 shillings made one pound, thus
- 240 pence per pound.

(There was also a _halfpenny_ that was half of one pence, but let's not get into that much detail in our game.)

Fortunately, even in this case we can default to the smallest unit of money when it comes to things like buying and selling.  By using the smallest unit of currency possible, we can do all calculations with integers instead of floats. For example, if a `Person` object with 5 pounds, 3 shillings and 5 pence wants to buy something that costs 1 shilling and 15 pence, we can calculate how much money the buyer has in pence:

- 5 pounds, which equals 5 * 240 = 1200 pence,
- 3 shillings, which equals 3 * 20 = 60 pence,
- 5 pence.

Which makes a total of 1265 pence for the buyer minus the item that costs 75 pence (3 shillings * 20, plus 15 pence). And since shopkeepers like to have small currencies (so they can give people change) and people like to have large currencies (so they aren't weighed down with too many coins), you can always subtract pence first, then shillings, and finally pounds when making a purchase.

There are many ways to represent money in a game. Let's think about them a bit:

- Simply adding properties like `pence`, `shilling` and `cent` to the `Person` type. This could work, but the `Person` type is already quite large. Plus there might be other objects that have money, such as `City` objects (if we wanted to keep track of a city's money supply). Even things like dead bodies could hold treasure and money.
- Creating an object type for types of currency with a link to an `owner`. This could make sense, considering that in life there are always heaps of money that are linked to us. For example, $1000.50 USD in a bank that belongs to you can be thought of as a `USD` type that holds 1000 `dollars` and 50 `cents` and has a link to you, the owner. The `USD` type could extend an abstract `Currency` type, and then you, a `Person` object, would have a backlink for all the piles of money that link to you. This might be too complicated for our game, however, which is centered on `PC`s and which doesn't have too many currencies to think about.
- Creating an abstract `HasMoney` type that can be extended by `Person` and anything else that holds money later on. This abstract type can hold each of the currencies as a property.

This last option seems easiest, so let's try that. We'll start with a scalar type called `Money` that we can use for every type of currency in the game, and give it a minimum value of 0:

```edgeql
  scalar type Money extending int64 {
    constraint min_value(0);
  }
```

With the `Money` type created, we can now move on to the abstract `HasMoney` type. Here it is to start:

```edgeql
  abstract type HasMoney {
    pounds: Money;
    shillings: Money;
    pence: Money;
    dollars: Money;
    cents: Money;
  }
```

Next, we can make all of these properties `required` and give them a default value of 0. This will make calculations easier since we will be guaranteed to always have a value instead of an empty set:

```edgeql
  abstract type HasMoney {
    required pounds: Money {
      default := 0;
    }
    required shillings: Money {
      default := 0;
    }
    required pence: Money {
      default := 0;
    }
    required cents: Money {
      default := 0;
    }
    required dollars: Money {
      default := 0;
    }
  }
```

And finally, to top it off let's add three computed properties called `total_pence`, `total_cents`, and `approx_wealth_in_pounds`. The first two represent the smallest units of currency in their respective countries, and so will represent a person's total wealth.

The last `approx_wealth_in_pounds` property shows the character's total wealth in pounds, which we will use as a general benchmark to compare all wealth. The calculation to make this property returns a float, but we are just using this for general comparisons and won't be using it to buy or sell anything. So we will cast it to an `int64` for readability.

At the time of the book, the exchange rate was about 8 US dollars to one pound (so 800 cents to one pound). Putting all that together, the final `HasMoney` type looks like this:

```edgeql
  abstract type HasMoney {
    required pounds: Money {
      default := 0;
    }
    required shillings: Money {
      default := 0;
    }
    required pence: Money {
      default := 0;
    }    
    required cents: Money {
      default := 0;
    }
    required dollars: Money {
      default := 0;
    }
    property total_pence := .pounds * 240 + .shillings * 20 + .pence;
    property total_cents := .dollars * 100 + .cents;
    property approx_wealth_in_pounds := <int64>.total_pence / 240 + .total_cents / 800;
  }
```

With all the above done, just add `extending HasMoney` to the `Person` type, and do a migration. Now it's time to give our characters some money!

An average laborer in the United Kingdom in 1893 made 30 pounds per year. Both Arthur Holmwood and Count Dracula are lords with almost unimaginable amounts of money, so we'll `update` them with random numbers that should be well beyond anything a laborer can earn in a lifetime:

```edgeql
update Person filter .name in { 'Arthur Holmwood', 'Count Dracula' }
 set {
   pounds := 3000 + <int64>(random() * 3000),
   shillings := 3000 + <int64>(random() * 3000),
   pence := 3000 + <int64>(random() * 3000)
};
```

And then a query to check how much money they have:

```edgeql
select Person {
  name,
  pounds,
  shillings,
  pence,
  total_pence } filter .name in { 'Arthur Holmwood', 'Count Dracula' };
```

The output should show each of these two characters with some pretty incredible amounts of money.

```
{
  default::NPC {
    name: 'Arthur Holmwood',
    pounds: 4249,
    shillings: 4219,
    pence: 3296,
    total_pence: 1107436,
  },
  default::Vampire {
    name: 'Count Dracula',
    pounds: 5967,
    shillings: 3539,
    pence: 3109,
    total_pence: 1505969,
  },
}
```

You can see that the `total_pence` property lets us quickly see who of the two is richer.

The other characters will get an `update` with more reasonable random amounts of money:

```edgeql
update Person filter .name not in { 'Arthur Holmwood', 'Count Dracula' }
 set {
   pence := 100 + <int64>(random() * 100),
   shillings := 20 + <int64>(random() * 100),
   pounds := 10 + <int64>(random() * 100)
 };
```

Then we have Quincy Morris, who should have some USD because he is an American. Back in 1890 an average American worker earned about $500 a year. He is pretty wealthy too so we'll `update` him with a few times that amount.

```edgeql
update Person filter .name = 'Quincey Morris'
 set { 
  dollars := 500 + <int64>(random() * 2000),
  cents := 500 + <int64>(random() * 2000)
};
```

And now let's try this query on all our `Person` objects.

```
select Person {
  name,
  total_pence,
  total_cents,
  approx_wealth_in_pounds
 } order by .approx_wealth_in_pounds desc;
```

The output should be similar to the output below, with Count Dracula and Arthur Holmwood on top, Quincey Morris in third place, and other `Person` objects in random order. Looks like the Crewmen and Innkeeper have been doing pretty well for themselves!

```
{
  default::Vampire {
    name: 'Count Dracula',
    total_pence: 1505969,
    total_cents: 0,
    approx_wealth_in_pounds: 6275,
  },
  default::NPC {
    name: 'Arthur Holmwood',
    total_pence: 1107436,
    total_cents: 0,
    approx_wealth_in_pounds: 4614,
  },
  default::NPC {
    name: 'Quincey Morris',
    total_pence: 27395,
    total_cents: 197076,
    approx_wealth_in_pounds: 360,
  },
  default::NPC {
    name: 'Mina Murray',
    total_pence: 27722,
    total_cents: 0,
    approx_wealth_in_pounds: 116,
  },
  default::Crewman {
    name: 'Crewman 2',
    total_pence: 25928,
    total_cents: 0,
    approx_wealth_in_pounds: 108,
  },
  default::Crewman {
    name: 'Crewman 4',
    total_pence: 25554,
    total_cents: 0,
    approx_wealth_in_pounds: 106,
  },
  default::NPC {
    name: 'The innkeeper',
    total_pence: 24409,
    total_cents: 0,
    approx_wealth_in_pounds: 102,
  },
  # And so on...
}
```

## Triggers

Let's give some thought to the actual users of our game - the people who will be signing up to make player characters to try to save the world from (or to help??) Count Dracula.

However the game takes shape, it will require some sort of account type to hold the information for the people using our game in real life. A simplified example of what we might need is as follows:

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

However, users tend to be fickle. They might delete their accounts and regret doing so, they might delete an account by mistake, and so on. In that case they will get into contact with us asking if an account can be restored. Legally, however, we can't just keep the `Account` and `CreditCardInfo` objects around just in case! We are obligated to delete all of a user's information when they ask. How to solve this issue?

An easy way to solve this problem is by using _triggers_, which were added in EdgeDB 3.0. A trigger represents some sort of action that we want to take place every time an object is inserted, updated, or deleted. Let's first take a look at the official syntax for triggers to get an idea of how to use them:

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
- Decide for which cases we want a trigger to happen (only after an `insert`, after either an `insert` or an `update`, etc.),
- Decide whether a trigger should happen on `each` object or once per query with `all`,
- And finally the expression of the trigger itself.

Inside a trigger we get access to the old object using `__old__` and the new object using `__new__`, though this depends on the operation. For example, when you `delete` there is no `__new__` object to access, nor is there an `__old__` object to access when doing an `insert`.

And now back to our `Account` type. To keep a minimal amount of information after a user's account is deleted, we can create a new type that holds this information. We can call this type `MinimalUserInfo`, because it will only hold their username and the `PC` objects they made. None of this information can be used to track down a person in real life so it is safe to hold, and at the same time it is enough information for us to restore an account. (Perhaps we will hold on to `PC` objects for 30 days or so in case a user changes their mind)

The `MinimalUserInfo` type is pretty simple, just a person's `username`, the `PC`s that they own, and an `int16` to hold a random number.

```sdl
type MinimalUserInfo {
  username: str;
  multi pcs: PC;
  passcode: int16;
}
```

And now let's add the trigger to `Account` that inserts a `MinimalUserInfo` object every time an `Account` object is deleted. With this trigger in place, we can freely delete any `Account` objects and the user's personal information will be removed: name, address, and credit card info. But a `MinimalUserInfo` will always be created at the same time which holds the `username` and the `PC` objects linked to it. It will also hold a `passcode` which holds a number in between 100 and 500.

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
    pcs := __old__.pcs,
    passcode := <int16>(random() * 400) + 100,
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
  passcode: int16;
}
```

Okay, let's do a migration and then insert an `Account` object. Look at all the personal information inside!

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
delete Account filter .user_name = 'deb_deb_999';
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
    passcode: 453,
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

With that we are holding none of Deborah's personal information anymore. All that we know is that some user called `deb_deb_999` had a `PC` called `LordOfSalty` - no personal info anywhere! Our software will check for new `MinimalUserInfo` objects and send off an email to users that have deleted their accounts to let them know how to restore their account if they wish. It would look something like this:

```
We're sorry to see you go...

Your personal information has been deleted from our database. Your PCs have been kept on file for the next 30 days should you choose to restore your account. Keep the passcode safe if you think you might want to restore your account! It's the only way to verify your identity now that all of your personal information has been removed.

Account name: deb_deb_999
PC names: LordOfSalty
Passcode: 453
```

So if Deborah decides that she wants to get back into the world of vampire gaming she can choose to restore her account - as long as she remembers her username and passcode.

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
