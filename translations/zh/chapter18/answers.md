# Chapter 18 Questions and Answers

#### 1. 在德古拉时代，德国的货币使用的是金马克（Goldmark）。一金马克是 100 芬尼（Pfennig）。你会如何制作这种货币类型？

这比 `Pound` 类型容易得多，因为比金马克小的单位只有芬尼（就像今天的美元和美分一样）。

```sdl
type Goldmark extending Currency {
  overloaded required major: str {
    default := 'Mark'
  }
  overloaded required minor: str {
    default := 'Pfennig'
  }
  overloaded required minor_conversion: int64 {
    default := 100
  }
}
```

#### 2. 尝试给上题中新增的类型添加两个注释（annotations）。其中一个称为 `description` 并说明 `One mark = 100 Pfennig`。另一个称为 `note`，并说明硬币的种类。

由于 `description` 已经是注释的一个选项了，我们不需要再为此做任何额外的声明。但是对于 `note`，我们还是需要声明：

`abstract annotation note;`

然后，我们将注释添加到 `Goldmark` 即可：

```sdl
type Goldmark extending Currency {
  annotation description := 'One Mark = 100 Pfennig';
  annotation note := 'Coin types: 1 Pfennig, 2 Pfennig, 5 Pfennig, 10 Pfennig, 20 Pfennig, 25 Pfennig';
  overloaded required major: str {
    default := 'Mark';
  };
  overloaded required minor: str {
    default := 'Pfennig';
  };
  overloaded required minor_conversion: int64 {
    default := 100;
  };
}
```

#### 3. 一个名叫戈德布兰（Godbrand）的吸血鬼刚刚袭击了一个村庄，将三个村民变成了 `MinorVampire`。你将如何一次插入涉及到的所有（7个）对象？

这是一个相当长的插入：

```edgeql
with new_vampires := {
    ('Fritz Frosch', '1850-01-15', '1893-09-11'),
    ('Levanta Sinyeva', '1862-02-24', '1893-09-11'),
    ('김훈', '1860-09-09', '1893-09-11')
  }
insert Vampire {
  name := 'Godbrand',
  slaves := (
    for new_vampire in new_vampires
    union (
      insert MinorVampire {
        name := 'Undead' ++ new_vampire.0,
        first_appearance := <cal::local_date>new_vampire.2,
        former_self := (
          insert NPC {
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
