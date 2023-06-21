---
tags: Complex Inserts, Schema Cleanup
---

# 第十八章 - 以牙还牙

> 范海辛（Van Helsing）是对的：米娜（Mina）与德古拉（Dracula）有关。他继续对米娜使用催眠术来了解德古拉的藏身之处以及他在做什么。乔纳森（Jonathan）对德古拉在伦敦的活动进行了大量的调查。他拜访了所有出售房子给德古拉的公司，以及一些曾为德古拉搬运过棺材的搬家公司。乔纳森越来越有信心，且从未停止过寻找德古拉的工作。终于，他们在伦敦找到了另一栋房子，里面存储着德古拉的全部财产。他们知道他一定会来拿，于是他决定等待他的到来……突然，德古拉跑进房子开始了攻击。乔纳森用刀砍了出去，割破了德古拉装满钱的包裹。德古拉抓起了一些掉落的钱便从窗户跳了出去。德古拉冲他们吼道：“你们每个人都会后悔的！你们以为你们让我无处可去；但其实我还有很多的落脚地。我的报复才刚刚开始！” 接着，他就消失了。

这里是个很好的提醒，我们可能该在游戏中引入“钱”的概念了。书中的角色们去过英国、罗马尼亚和德国等国家，他们每个人都有自己的钱。“抽象类型”在这里似乎是一个不错的选择：我们应该创建一个 `abstract type Currency`，之后可以将其扩展为所有其他类型的货币。

现在，有一个困难是：在 19 世纪（1800 年代），货币体系比今天更复杂。例如，在英国，不是 100 便士兑换 1 磅，而是：

- 12 便士（最小单位的硬币）兑换一先令，
- 20 先令等于一磅，因此
- 每磅有 240 便士。

（还有一个 _半便士铜币（halfpenny）_ 是一便士的一半，但这里我们就不在游戏中引入这么多细节了。）

为了说明上面的规则，我们给 `Currency` 定义了三个属性（货币单位）：`major`、`minor` 和 `sub_minor`，且它们每一个都会有一个金额，其中后两个还会有一个用于兑换换算的数字；此外，`Currency` 还包含一个 `owner: Person` 来指明钱的拥有者。所以 `Currency` 的定义如下所示：

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

你会注意到只有属性 `major` 是 `required` 的，因为有些货币甚至没有美分之类的东西。在现代，包括日元、韩元等只是一个单一的货币单位加一个数字。

我们还对所有 `*_amount` 属性添加了 `min_value(0)` 的约束，这样书中的角色们就不能透支他们的账户了。还有一些复杂的事情可以做，比如信用和负货币，但现在我们暂时先忽略。

然后来看一下我们的第一种货币：`Pound` 类型。其属性 `minor` 称为 `'shilling'`，同时我们使用 `minor_conversion` 来说明获取 1 磅所需的金额（是 20 shilling）。`'pence'` 也一样。于是，我们的角色可以开始收集各种硬币了，但最终的总价值仍然可以很快地转换成英镑来表示。下面是 `Pound` 类型的定义：

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

现在，让我们来给德古拉分配一些钱：2500 英镑、50 先令和 200 便士。在 1893 年这也许是一大笔钱了。

```edgeql
insert Pound {
  owner := (select Person filter .name = 'Count Dracula'),
  major_amount := 2500,
  minor_amount := 50,
  sub_minor_amount := 200
};
```

然后我们可以通过使用转换率进行计算，并以“镑”为单位显示他所拥有的总金额：

```edgeql
select Currency {
  owner: {name},
  total := .major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)
};
```

他拥有这么多：

```
{default::Pound {owner: default::Vampire {name: 'Count Dracula'}, total: 2503.3333333333335}}
```

我们从之前的章节中已经了解到亚瑟（Arthur）现在是戈达尔明勋爵（Lord Godalming）了，他同样拥有一大笔财富，至于其他人，我们并不确定。现在让我们给这些人分配一些随机数量的钱，同时 `select` 它以显示随机到的结果。对于随机数，我们将使用我们之前用于 `strength` 的方法：`round()` 一个 `random()` 数并乘以最大值。

但我们希望在显示每个人的金钱总数时，可以将其转换为 `decimal` 类型。这样我们就可以控制其显示为 555.76 而不是 555.76545256。为此，我们仍然使用 `round()` 函数，但使用其最后一个签名：

```sdl
std::round(value: int64) -> float64
std::round(value: float64) -> float64
std::round(value: bigint) -> bigint
std::round(value: decimal) -> decimal
std::round(value: decimal, d: int64) -> decimal
```

最后一个签名有一个额外的 `d: int64` 部分，用于表示我们想要取到的小数位数。

总之，代码如下所示：

```edgeql
select (for character in {'Jonathan Harker', 'Mina Murray', 'The innkeeper', 'Emil Sinclair'}
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

结果如下所示，其中每份钱都有一个所有者：

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

（如果你不想看到 `decimal` 类型最后的 `n`，只需将其转换为 `<float32>` 或 `<float64>` 即可）

你现在也许会注意到，关于如何显示金钱可能存在一些争论。它应该是一个 `Currency` 并链接到所有者吗？或者它应该是一个 `Person` 并链接到名为 `money` 的属性？前者（我们的方法）对于现实游戏来说可能更简单，因为游戏中存在多种 `Currency` 类型。如果我们选择另一种方法，我们将需要给 `Person` 类型创建很多个链接，分别链接到不同类型的货币，并且大多数都为零。而使用我们的方法，我们只需要在角色开始拥有某种货币时为其创造“成堆”的金钱。或者这些“堆”可能是钱包和袋子之类的东西，如果游戏中的角色可能会丢失这些“成堆”的钱，我们可以将 `required owner: Person;` 改为 `optional owner: Person;`。

当然，如果我们的游戏世界里只有一种类型的钱，那么将它放在 `Person` 类型中会更加简单。下面的内容并没有打算真的修改架构，只是一起来想象一下该如何做到这一点。假设游戏只发生在美国境内，那么在没有 `Currency` 这个抽象类型的情况下，像下面这样做会更容易：

```sdl
type Dollar {
  required dollars: int64;
  required cents: int64;
  property total_money := .dollars + (.cents / 100)
}
```

顺便说一下，由于 `/ 100` 部分，`total_money` 的类型将变为 `float64`。我们可使用下面的语句进行快速的验证：

```edgeql
select (100 + (55 / 100)) is float64;
```

结果是：`{true}`。

当我们进行插入并使用 `select` 检查 `total_money` 属性时，我们可以看到相同的效果：

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

输出是：`{default::Dollar {total_money: 100.55}}`。完美！

然后，我们可以给 `Person` 类型创建一个新的链接 `money` 指向类型 `Dollar`。注意：在我们的架构中，并不是真的需要这种 `Dollar` 类型，如果需要，它也会是 `type Dollar extending Currency`。

最后一个注意事项：我们的 `total_money` 属性只是通过除以 100 创建的，因此它以小数有限的方式使用了 `float64`（这很好）。但是你要小心浮点数，因为它们并不总是精确的，例如，如果我们需要除以 3，我们会得到类似 `100 / 3 = 33.33333333` 的结果……这对于实际货币来说并不合适。因此，在这种情况下，最好坚持使用整数。

## 清理架构

我们已经接近本书的结尾了，是时候开始清理架构并插入一些内容了。

首先，原有架构里有两条对 `City` 的插入：

```edgeql
insert City {
  name := 'Munich',
};

insert City {
  name := 'London',
};
```

现在，我们将其改为使用 `for` 循环的一次性插入：

```edgeql
for city_name in {'Munich', 'London'}
union (
  insert City {
    name := city_name
  }
);
```

然后，我们将对插入的四个 `Country` 对象（匈牙利、罗马尼亚、法国、斯洛伐克）执行相同的操作。如下所示：

```edgeql
for country_name in {'Hungary', 'Romania', 'France', 'Slovakia'}
union (
  insert Country {
    name := country_name
  }
);
```

其他 `City` 的插入有点不同：一些有 `modern_name`，一些有 `population`。在真正的游戏中，我们会以下面这种形式（使用元组）一次性将它们全部插入：

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

对所有的 `NPC` 类型，它们的 `first_appearance` 数据等，我们可以做同样的处理。但在本教程中没有那么多的城市和角色需要插入，所以我们还不需要那么的系统。

我们还可以进一步整合下面的语句：

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

都放入 `Ship` 的插入当中：

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

看起来确实好多了！

[→ 点击这里查看到第 18 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 在德古拉时代，德国的货币使用的是金马克（Goldmark）。一金马克是 100 芬尼（Pfennig）。你会如何制作这种货币类型？

2. 尝试给上题中新增的类型添加两个注释（annotations）。其中一个称为 `description` 并说明 `One mark = 100 Pfennig`。另一个称为 `note`，并说明硬币的种类。

   [这里是硬币的种类](https://en.wikipedia.org/w/index.php?title=German_gold_mark&oldid=972733514#Base_metal_coins)：1, 2, 5, 10, 20, 25 芬尼（Pfennig）硬币。

3. 一个名叫戈德布兰（Godbrand）的吸血鬼刚刚袭击了一个村庄，将三个村民变成了 `MinorVampire`。你将如何一次插入涉及到的所有（7个）对象？

   下面是他们的数据（姓名、出生日期（`first_appearance`）、变成 MinorVampire 的日期（`last_appearance`））：

   ```
   ('Fritz Frosch', '1850-01-15', '1893-09-11'),
   ('Levanta Sinyeva', '1862-02-24', '1893-09-11'),
   ('김훈', '1860-09-09', '1893-09-11'),
   ```

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _只有米娜可以告诉他们德古拉去了哪里。_
