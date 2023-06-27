---
tags: Defaults, Overloading, For Loops
---

# 第九章 - 在英格兰发生的奇怪事

在本章我们回到了几周前，即“德米特号”（载有德古拉的船）刚刚离开瓦尔纳（Varna），而米娜（Mina）和露西（Lucy）还没有启程前往惠特比（Whitby）的时候。故事情节也将分为两部分介绍。这里是第一个：

> 我们仍然不知道乔纳森（Jonathan）在哪里，德米特号（The Demeter）正在前往英格兰（England）的途中，德古拉（Dracula）就在船上。与此同时，米娜（Mina）正在伦敦给她的朋友露西·韦斯特拉（Lucy Westenra）写信。露西有三个男朋友，分别是约翰·苏厄德医生（Dr. John Seward）、昆西·莫里斯（Quincey Morris）和亚瑟·霍姆伍德（Arthur Holmwood），他们都爱慕着露西并想与她结婚……

## 关于日期的更多处理

按照故事情节的发展，看起来我们还有更多人物需要插入创建。但在此之前，让我们再来斟酌一下叫做“德米特号”的那艘船。船上所有人都被德古拉（Dracula）杀死了，但我们并不想删除船员，因为他们仍然是我们游戏的一部分。小说告诉我们，这艘船是在 7 月 6 日离开瓦尔纳的，船上的最后一个人（船长）死于 8 月 4 日（1893 年）。

这正是给 `Person` 类型添加两个新属性以展示一个角色在游戏里存活时间的好时机。我们给它们分别命名为 `first_appearance` 和 `last_appearance`。`last_appearance` 比起 `death` 更为合适，因为角色是否生理性死亡对我们的游戏来说无关紧要：我们只想知道角色是否还会出现在游戏当中。

对于这两个属性，为了简单起见，我们将使用 `cal::local_date`。虽然也可以使用包含了时间的 `cal::local_datetime` 类型，但目前我们只用精确到日期。（当然还有 `cal::local_time` 类型，但它只是用来表达一天中的某个时间。）

现在，如果对具有 `first_appearance` 和 `last_appearance` 属性的 `Crewman` 对象进行插入，应按如下所示：

```edgeql
insert Crewman {
  number := count(detached Crewman) +1,
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16),
};
```

由于我们已经插入了很多 `Crewman` 对象，如果我们假设他们都同时死亡（这里并不需要那么精确），我们即可以很轻松地使用 `update` 和 `set` 对这些对象进行更新。

由于 `cal::local_date` 需要的格式非常简单：即：YYYYMMDD，因此在插入中使用它的最简单方法就是直接对字符串进行转换：

```edgeql
select <cal::local_date>'1893-07-08';
```

但是我们之前遇到过将年、月、日以单独数字作为输入的相关函数，考虑到可读性，这里我们将继续使用类似的函数。

之前我们使用的是带有七个参数的函数 `std::to_datetime`；这里我们将使用类似但更短的函数 {eql:func}`docs:cal::to_local_date`。它只需要三个整数。

下面是它的三个签名（我们将使用第三个）：

```
cal::to_local_date(s: str, fmt: optional str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

现在，我们来更新 `Crewman` 对象并赋予它们相同的起止日期：

```edgeql
update Crewman
set {
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16)
};
```

当然这些日期的具体数值应该取决于我们的游戏情节。比如，一个 `PC` 可以在这艘船航行至英格兰的途中登船访问吗？在德古拉杀死船员之前，会有试图拯救船员的任务吗？如果出现类似的故事情节，那么我们将需要更为精确的日期，甚至是时间。但现在，这些大致的日期就足够了。

## 添加默认值并使用重载

现在让我们回到对新角色的插入。首先，我们将创建露西（Lucy）：

```edgeql
insert NPC {
  name := 'Lucy Westenra',
  places_visited := (select City filter .name = 'London')
};
```

嗯，看起来每当我们添加一个角色，我们都要为插入 'London' 而写很多代码。我们还剩下三个角色，他们也都来自伦敦。为了节省一些工作，我们可以将伦敦设为 `NPC` 的 `places_visited` 的默认值。为此，我们需要做两件事：用 `default` 声明默认值，以及使用关键字 `overloaded` 进行重载。`overloaded` 这个词表明我们使用 `places_visited` 的方式将不同于我们扩展的 `Person` 类型。

添加了 `default` 和 `overloaded` 后，`NPC` 的定义如下所示：

```sdl
type NPC extending Person {
  age: HumanAge;
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

## datetime_current()

这里介绍一个方便实用的的函数，即 {eql:func}` ``datetime_current()`` <docs:std::datetime_current>`，它可以给出当前的日期时间。让我们试试看：

```
db> select datetime_current();
{<datetime>'2020-11-17T06:13:24.418765000Z'}
```

如果你在插入对象时需要一个发布日期，这个函数会很有用。有了发布时间，你可以按日期排序，如果有重复项，则删除最近插入的条目，等等。让我们想象一下如果我们把它放在 `Place` 类型中会是什么样子。如下所示：

```sdl
abstract type Place {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  important_places: array<str>;
  property post_date := datetime_of_statement(); # this is new
}
```

但其实这不是我们想要的，因为这样做只会在我们 *查询* `Place` 对象时生成日期，而不是我们所期待的在插入时生成对应日期。如果要创建一个带有插入日期的 `Place` 类型，我们应该使用 `default`：

```sdl
abstract type Place {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  important_places: array<str>;
  post_date: datetime {
    default := datetime_of_statement()
  }
}
```

在我们的架构（schema）中我们并不需要这个日期，所以我们并不打算真的改变 `Place` 的定义，这里只是为了展示你可以如何操作。

## 使用 for 和 union

现在，我们几乎准备好插入那些新角色了，我们不再需要每次都编写 `(select City filter .name = 'London')`。但是，如果我们可以只使用单个插入而不是三个类似的插入不是更好吗？

要做到这一点，我们可以使用一个 `for` 循环，后面跟着关键字 `union`。首先，这是 `for` 的部分：

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
```

换句话说就是：获取这三个字符串组成的集合，并对每个字符串做一些事情。`character_name` 是我们选择调用集合中每个字符串时所用的变量名称。

`union` 是用于将集合连接在一起的关键字。例如：

```edgeql
with city_names := (select City.name),
  castle_names := (select Castle.name),
select city_names union castle_names;
```

该查询将 `city_names` 和 `castle_names` 两个名称集合合并在一起进行输出：`{'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Castle Dracula'}`。

现在让我们回到带有变量名 `character_name` 的 `for` 循环，如下所示：

```edgeql
for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
union (
  insert NPC {
    name := character_name,
    lovers := (select Person filter .name = 'Lucy Westenra'),
  }
);
```

执行后，我们会得到三个 `uuid`，说明三个角色均已被插入。

然后，让我们来检查一下，以确保我们的操作确实成功了：

```edgeql
select NPC {
  name,
  places_visited: {
    name,
  },
  lover: {
    name,
  },
} filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'};
```

结果正如我们所希望的那样，他们现在都与露西有关（即他们的 `lover` 都是露西）。

```
{
  default::NPC {
    name: 'John Seward',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
  default::NPC {
    name: 'Quincey Morris',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
  default::NPC {
    name: 'Arthur Holmwood',
    places_visited: {default::City {name: 'London'}},
    lover: {default::NPC {name: 'Lucy Westenra'}},
  },
}
```

因为John Seward是医生，所以我们会给他一个叫'Dr'的 `title`:

```edgeql
update NPC filter .name = 'John Seward'
set { title := 'Dr.' };
```

顺便说一下，现在我们也可以使用同样的方法将我们的五个 `Crewman` 对象用一个 `insert` 完成插入，而不是 `insert` 五次。我们可以将船员的编号放在一个集合中，并使用 `for` 和 `union` 来创建他们。当然，在这之前我们已经使用了 `update` 对他们进行了更改，但从现在开始，在我们的代码中，船员的插入将如下所示：

```edgeql
for n in {1, 2, 3, 4, 5}
union (
  insert Crewman {
    number := n,
    first_appearance := cal::to_local_date(1893, 7, 6),
    last_appearance := cal::to_local_date(1893, 7, 16),
  }
);
```

使用 `for` 时，可以先熟悉一下 {ref}`要遵循的书写顺序 <docs:ref_eql_statements_for>`：

```edgeql-synopsis
[ with with-item [, ...] ]

for variable in iterator-expr

union output-expr ;
```

最重要的部分是 *iterator-expr*，它需要一个简单的表达式，返回某个集合。通常是置于 `{` 和 `}` 中的一个集合。也可以是返回集合的一个路径，例如 `NPC.places_visited`，也可以是返回集合的一个函数调用，例如 `array_unpack()`。对于更复杂的表达来说，要放置于圆括号中引用。

现在是时候更新露西（Lucy）的情人链接了（但她有三个情人）。露西已经破坏了我们将 `lover` 仅仅作为一个 `link`（即 `single link`）的设定。我们不得不将其变更为 `multi` 链接，这样我们就可以同时添加他们三个人给露西了。这里是我们对露西的更新：

```edgeql
update NPC filter .name = 'Lucy Westenra'
set {
  lovers := (
    select Person filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};
```

现在，我们用查询来验证对她的更新是否有效。这次我们会在过滤器中使用 `like`：

```edgeql
select NPC {
  name,
  lover: {
    name
  }
} filter .name like 'Lucy%';
```

结果确实输出了露西和她的三个情人：

```
{
  default::NPC {
    name: 'Lucy Westenra',
    lover: {
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
    },
  },
}
```

## 用重载替代新类型的创建

我们已经了解到了关键字 `overloaded`，因此，我们可能不再需要 `NPC` 中用到的 `HumanAge` 类型了。现在的 `HumanAge` 类型长这样：

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

你应该还记得我们制作这个类型是因为吸血鬼可以永生，但人类只能活到 120 岁。现在我们来对其进行简化。首先，我们将 `age` 属性移到 `Person` 类型。然后在 `NPC` 类型内使用 `overloaded` 对 `age` 添加一个约束。现在 `NPC` 里使用了两个 `overloaded`：

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120);
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

这很方便，因为我们也可以从 `Vampire` 中删除 `age` 了：

```sdl
type Vampire extending Person {
  # age: int16; **Delete this one now**
  multi slaves: MinorVampire;
}
```

由此可见，如果你可以正确地使用抽象类型和关键字 `overloaded`，你的架构是可以被简化许多的。

接下来，让我们继续阅读本章故事内容的剩余部分。它解释了露西（Lucy）在做什么：

> ……露西选择了嫁给亚瑟·霍姆伍德（Arthur Holmwood），并向另外两人道了歉。另外两个男人很难过，但幸运的是三个男人彼此成为了朋友。其中，苏厄德医生（Dr. Seward）很沮丧，并试图专注于他的工作以摆脱情伤。他是一名精神病医生，在伦敦郊外不远处的精神病院工作，附近有一座名为 Carfax 的大别墅。疯人院里有个奇怪的人，名叫伦菲尔德（Renfield），苏厄德医生对他很感兴趣。伦菲尔德有时冷静，有时癫狂，苏厄德医生不清楚他的情绪变化为何如此之快。此外，伦菲尔德似乎认为吃活的东西可以获得力量。他并不是吸血鬼，但有时看起来又很相似。

哎呀！看起来露西（Lucy）已经没有三个情人了。现在我们必须将她更新为只有亚瑟（Arthur）一个情人了：

```edgeql
update NPC filter .name = 'Lucy Westenra'
set {
  lovers := (select detached NPC filter .name = 'Arthur Holmwood'),
};
```

然后，将她从另外两个男人的 `lover` 中移除——我们只好给他们一个悲伤的空集了。

```edgeql
update NPC filter .name in {'John Seward', 'Quincey Morris'}
set {
  lovers := {} # 😢
};
```

现在，基本上一切都是最新的了。就剩下为神秘的伦菲尔德（Renfield）创建数据了。这很容易，因为他没有情人，不需要做 `filter`：

```edgeql
insert NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1893, 5, 26),
  strength := 10,
};
```

但他与德古拉似乎有某种关系，类似于 `MinorVampire` 类型但又不同。他也很强壮（稍后我们会看到），所以我们给他的 `strength` 设置为 10。稍后我们将对他以及他与德古拉的关系有更进一步的探索。

[→ 点击这里查看到第 9 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 为什么下面这个插入不起作用，该如何修复？

   ```edgeql
   for castle in ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
   union (
     insert Castle {
       name := castle
     }
   );
   ```

2. 如何在显示城堡名称的同时进行与上题相同的插入？
3. 如果所有的吸血鬼都需要一个最小为 10 的力量值，如何修改 `Vampire` 类型？
4. 如何更新所有的 `Person` 类型的对象，表明他们都死于 1893 年 9 月 11 日？

   提示：下面是 `Person` 类型当前的定义：

   ```sdl
   abstract type Person {
     required name: str {
       delegated constraint exclusive;
     }
     age: int16;
     strength: int16;
     multi places_visited: Place;
     multi lovers: Person;
     first_appearance: cal::local_date;
     last_appearance: cal::local_date;
   }
   ```

5. 如果所有名字中带有 `e` 或 `a` 的 `Person` 角色都被复活了，你将如何更新？

   提示：“复活”意味着 `last_appearance` 应该返回 `{}`。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _大雾和风暴袭击了惠特比市。_
