# Chapter 18 Questions and Answers

#### 1. During the time of Dracula, the Goldmark was used in Germany. One Goldmark had 100 Pfennig. How would you make this type?

This is a lot easier than the `Pound` type, since the only thing smaller than a Goldmark was a Pfennig (as easy as dollars/euros and cents today).

```sdl
type Goldmark extending Currency {
  overloaded required property major {
    default := 'Mark'
  }
  overloaded required property minor {
    default := 'Pfennig'
  }
  overloaded required property minor_conversion {
    default := 100
  }
}
```

#### 2. Try adding two annotations to this type. One should be called `description` and mention that `One mark = 100 Pfennig`. The other should be called `note` and mention the types of coins there are.

Since `description` is already an option for annotations we wouldn't need to do anything for that. But for `note`, we would need to add this:

`abstract annotation note;`

After that, we add the annotations to Goldmark and it's done:

```sdl
type Goldmark extending Currency {
  annotation description := 'One Mark = 100 Pfennig';
  annotation note := 'Coin types: 1 Pfennig, 2 Pfennig, 5 Pfennig, 10 Pfennig, 20 Pfennig, 25 Pfennig';
  overloaded required property major {
    default := 'Mark';
  };
  overloaded required property minor {
    default := 'Pfennig';
  };
  overloaded required property minor_conversion {
    default := 100;
  };
}
```

#### 3. A vampire named Godbrand has just attacked a village and turned three villagers into `MinorVampire`s. How would you insert all four of them at once?

Here it is, a fairly long insert:

```edgeql
WITH new_vampires := {
    ('Fritz Frosch', '1850-01-15', '1887-09-11'),
    ('Levanta Sinyeva', '1862-02-24', '1887-09-11'),
    ('김훈', '1860-09-09', '1887-09-11')
  }
INSERT Vampire {
  name := 'Godbrand',
  slaves := (
    FOR new_vampire IN new_vampires
    UNION (
      INSERT MinorVampire {
        name := 'Undead' ++ new_vampire.0,
        first_appearance := <cal::local_date>new_vampire.2,
        former_self := (
          INSERT NPC {
            name := new_vampire.0,
            first_appearance := <cal::local_date>new_vampire.1,
            last_appearance := <cal::local_date>new_vampire.2
          }
        )
      }
    )
  )
};
```
