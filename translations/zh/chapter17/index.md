---
tags: Aliases, Named Tuples
---

# 第十七章 - 可怜的伦菲尔德和米娜

> 上一章中，苏厄德医生（Dr. Seward）和范海辛医生（Dr. Van Helsing）想过放伦菲尔德（Renfield）出去，可他们还是无法信任他。但事实证明，伦菲尔德说的是实话！那天晚上，德古拉（Dracula）发现了他们正在摧毁他的棺材，于是他决定进行报复并袭击了米娜（Mina）。德古拉成功了，米娜正慢慢地变成一只吸血鬼。虽然她现在仍然是人类，但她与德古拉已经开始有了联系。

> 此时，大家发现伦菲尔德躺在血泊中，奄奄一息。伦菲尔德感到很抱歉，并说出了真相。他和德古拉一直保持着联系，他以为德古拉可以帮助他变成吸血鬼，所以他让德古拉进了屋子。但是一进屋，德古拉就不再理他，而是径直走向了米娜的房间。伦菲尔德试图攻击德古拉以阻止他伤害米娜，但德古拉太强壮了，他打不过。

> 不过，米娜并没有放弃与德古拉的战斗，并提出了一个好主意。如果她现在与德古拉有了联系，那么当范海辛（Van Helsing）对她使用催眠术时，会发生什么呢？这可行吗？范海辛拿出怀表对米娜说：“请专心看着这只表。你正在感到困倦……你有什么感觉？想想那个袭击你的人，试着去感受他在哪里……” 

## 命名元组

还记得我们在第 12 章中创建的 `fight()` 函数吗？它被重载后可以接受 `(Person, Person)` 或 `(str, int16, str)` 作为输入。现在让我们将德古拉（Dracula）和伦菲尔德（Renfield）输入进去：

```edgeql
with
  dracula := (select Person filter .name = 'Count Dracula'),
  renfield := (select Person filter .name = 'Renfield'),
select fight(dracula, renfield);
```

毫无疑问，结果当然会是 `{'Count Dracula wins!'}`。

执行同样查询的另一种方法是使用单个元组。然后我们可以将指向德古拉的 `.0` 和指向伦菲尔德的 `.1` 输入给函数，如下所示：

```edgeql
with fighters := (
    (select Person filter .name = 'Count Dracula'),
    (select Person filter .name = 'Renfield')
  ),
select fight(fighters.0, fighters.1);
```

看起来还不错，但有一种方法可以使它更清楚：我们可以为元组中的项目命名，来替代使用 `.0` 和 `.1`。“命名”的语句看起来像是给变量赋值一个普通的计算（computed）链接或属性，同样使用 `:=`：

```edgeql
with fighters := (
    dracula := (select Person filter .name = 'Count Dracula'),
    renfield := (select Person filter .name = 'Renfield')
  ),
select fight(fighters.dracula, fighters.renfield);
```

下面是命名元组的另一个示例：

```edgeql
with minor_vampires := (
    women := (select MinorVampire filter .name like '%Woman%'),
    lucy := (select MinorVampire filter .name like '%Lucy%')
  ),
select (minor_vampires.women.name, minor_vampires.lucy.name);
```

输出是（别忘了笛卡尔乘法）：

```
{('Vampire Woman 1', 'Lucy'), ('Vampire Woman 2', 'Lucy'), ('Vampire Woman 3', 'Lucy')}
```

伦菲尔德已经不在人世了，所以我们需要使用 `update` 给他的 `last_appearance` 更新一个日期。让我们再来体验一次在更新的同时，使用 `select` 显示“更新”结果的奇妙：

```edgeql
select ( # Put the whole update inside
  update NPC filter .name = 'Renfield'
  set {
    last_appearance := <cal::local_date>'1893-10-03'
  }
) # then use it to call up name and last_appearance
{
  name,
  last_appearance
};
```

结果是：`{default::NPC {name: 'Renfield', last_appearance: <cal::local_date>'1893-10-03'}}`

最后要提的是：在元组中命名一个项目对项目本身是没有任何影响的。所以：

```edgeql
select ('Lucy Westenra', 'Renfield') = (character1 := 'Lucy Westenra', character2 := 'Renfield');
```

返回 `{true}`。

## 将抽象类型放在一起

哪里有吸血鬼，哪里就有吸血鬼猎人。有时猎人会摧毁棺材，有时吸血鬼会建造更多。如果能有一个通用的方式来更新这个信息就更好了，但现在的问题是：

- `HasCoffins` 类型是一个抽象类型，只有具有一个属性：`coffins`，
- 可以放置棺材的地方是 `Place` 及其扩展出的所有类型，还有 `Ship`，
- 最好的过滤方式是通过 `.name`，但是 `HasCoffins` 没有这个属性。

所以也许我们可以把这个类型改成一个叫做 `HasNameAndCoffins` 的类型，然后把 `name` 和 `coffins` 两个属性都放进去，因为在我们游戏中每一个地方都需要一个名字和可能被放置的棺材数量。请记住，拥有 0 个棺材的地方意味着吸血鬼不能在此久留：他只能在夜间、太阳升起前快速行动。

下面是具有新属性 `name` 的新类型。我们会给 `name` 添加两个约束：`exclusive` 和 `max_len_value` 以防止重名及名称过长。

```sdl
abstract type HasNameAndCoffins {
  required coffins: int16 {
    default := 0;
  }
  required name: str {
    delegated constraint exclusive;
    constraint max_len_value(30);
  }
}
```

然后，我们就可以更改我们的 `Ship` 类型了（注意我们删除了 `name`）

```sdl
type Ship extending HasNameAndCoffins {
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

还有 `Place` 类型。现在看起来简单多了：

```sdl
abstract type Place extending HasNameAndCoffins {
  modern_name: str;
  important_places: array<str>;
}
```

最后，我们来更改我们的 `can_enter()` 函数。之前它需要一个 `HasCoffins` 类型作为其中一个输入参数：

```sdl
function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := (select Person filter .name = person_name),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.' if has_coffins else vampire.name ++ ' cannot enter.'
    );
```

但现在 `HasNameAndCoffins` 包含了 `name`，所以我们可以通过输入地点的字符串名称来替代之前的参数，将函数更改为：

```sdl
function can_enter(person_name: str, place: str) -> optional str
  using (
    with
      vampire := assert_single(
        (select Person filter .name = person_name)
      ),
      enter_place := assert_single(
        (select HasNameAndCoffins filter .name = place)
      )
    select vampire.name ++ ' can enter.' if enter_place.coffins > 0 else vampire.name ++ ' cannot enter.'
  );
```

现在我们输入 `can_enter('Count Dracula', 'Munich')` 则可以得到 `'Count Dracula cannot enter.'`。这是合理的：因为德古拉没有带任何棺材到慕尼黑（Munich）。

最后，我们可以通过通用更新来更改棺材的数量。这很容易：

```sdl
update HasNameAndCoffins filter .name = <str>$place_name
set {
  coffins := .coffins + <int16>$number
}
```

现在，让我们用上面的方法给 `The Demeter`（德古拉前往伦敦时乘坐的船）上放一些棺材：

```
db> update HasNameAndCoffins filter .name = <str>$place_name
....... set {
.......   coffins := .coffins + <int16>$number
....... };
Parameter <str>$place_name: The Demeter
Parameter <int16>$number: 10
```

然后让我们通过查询来检查其执行结果：

```edgeql
select Ship {
  name,
  coffins,
};
```

结果是：`{default::Ship {name: 'The Demeter', coffins: 10}}`。即，德米特号（The Demeter）得到了 10 个棺材。

## 用别名“创建”子类型

我们在本书中使用了大量的抽象类型。你会注意到抽象类型本身通常是由非常普遍的概念构成的：`Person`、`HasNameAndCoffins` 等等。在现实生活中的数据库里，你可能会以 `HasEmail`、`HasID` 等形式看到它们，并由此扩展出子类型。别名（Aliases）也可以“创建”子类型（但不是真的创建），使用的是 `:=` 而不是 `extending`，且无法“继承”多个类型。

现在就让我们在我们的架构（schema）中试着运用一下别名。先来看一下德米特号（The Demeter），船从保加利亚（Bulgaria）的瓦尔纳（Varna）出发，最终抵达伦敦（London）。现在假设我们在游戏中已经将瓦尔纳建成了一个可供游戏角色探索的大港口，并且正在尝试改变架构以反映这一点。目前，`Crewman` 类型的定义是：

```sdl
type Crewman extending HasNumber, Person {
}
```

想象一下，出于某种原因，我们需要一个 `CrewmanInBulgaria` 别名，因为保加利亚人互相称对方为“Gospodin”而不是“Mr.”，我们的游戏需要反映出这一点。即，无论何时我们的船员出现在保加利亚时，都将被称为“Gospodin (+名字)”。此外，我们需要再添加一个计算（computed）链接 `current_location`，并链接到名为保加利亚的 `Place`。如下所示：

```sdl
alias CrewmanInBulgaria := Crewman {
  name := 'Gospodin ' ++ .name,
  current_location := (select Place filter .name = 'Bulgaria'),
};
```

你可能马上注意到了别名（alias）里面的 `name` 和 `current_location` 被逗号隔开了，而不是分号。它表明这不是在“真的”创建新类型：它只是在现有的 `Crewman` 类型之上创建了一个 [_shape_](https://www.edgedb.com/docs/edgeql/select#shapes)。出于同样的原因，你不能执行 `insert CrewmanInBulgaria`，因为这样的类型并不真实存在，错误提示为：

```
error: cannot insert into expression alias 'default::CrewmanInBulgaria'
```

所以，所有的插入仍然是通过 `Crewman` 类型完成的。但是因为别名（alias）是一个子类型（subtype）和一个形状（shape），所以我们可以像选择其他任何类型一样选择它。现在，我们先来添加保加利亚：

```edgeql
insert Country {
  name := 'Bulgaria'
};
```

然后，选择这个新创建的别名看看我们能得到什么：

```edgeql
select CrewmanInBulgaria {
  name,
  current_location: {
    name
  }
};
```

我们可以看到，输出对象的类型仍然是 `Crewman`，但他们的名字中添加了 _Gospodin_ 并链接到了我们刚刚插入的 `Country` 对象。

```
{
  default::Crewman {
    name: 'Gospodin Crewman 1',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 2',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 3',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 4',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 5',
    current_location: default::Country {name: 'Bulgaria'},
  },
}
```

此外，{ref}`关于别名的文档 <docs:ref_cheatsheet_aliases>` 中提到，别名允许你使用“来自 GraphQL 的 EdgeQL（表达式、聚合函数、反向链接导航）的全部功能”，所以如果你经常使用 GraphQL，请记住别名。

## 本地类型别名

当我们在编写 `alias CrewmanInBulgaria := Crewman` 时，我们只是使用了 `:=` 来声明我们的别名。那么我们是否可以在查询中做类似的事情呢？答案是肯定的：我们可以通过使用 `with` 为现有类型指定一个新名称。（实际上，我们一直在使用的关键字 `with` 就是表达一个“{ref}`用于定义别名的块 <docs:ref_eql_with>`”）。以下面这个简单的查询为例，它显示了德古拉伯爵和他的奴隶们的名字：

```edgeql
select Vampire {
  name,
  slaves: {
    name
  }
};
```

如果我们想使用 `with` 创建一个与 `Vampire` 相同的新类型，我们可以这样做：

```edgeql
with Drac := Vampire,
select Drac {
  name,
  slaves: {
    name
  }
};
```

到目前为止，没什么特别的，因为输出是相同的：

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Vampire Woman 1'},
      default::MinorVampire {name: 'Vampire Woman 2'},
      default::MinorVampire {name: 'Vampire Woman 3'},
      default::MinorVampire {name: 'Lucy'},
    },
  },
}
```

但当我们要使用这种新类型对创建它的类型进行操作或比较时，这个别名则变得十分有用。它的作用与 `detached` 相同，但因为我们给它起了一个名字，所以使用起来更加灵活易读。

现在，让我们来试一下。假设我们正在测试我们的游戏引擎，需要对函数 `fight()` 进行大量的测试。`fight()` 当前的定义很简单（实际中可能会很复杂）：

```sdl
function fight(one: Person, two: Person) -> str
  using (
    one.name ++ ' wins!' if one.strength > two.strength else
    two.name ++ ' wins!'
  );
```

出于调试目的，最好能显示出更多详细的信息。因此，我们创建了类似的、名为 `fight_2()` 的函数，并添加了更多关于谁与谁战斗的具体信息。

```edgeql
create function fight_2(one: Person, two: Person) -> str
  using (
    one.name ++ ' fights ' ++ two.name ++ '. ' ++ one.name ++ ' wins!'
    if one.strength > two.strength else
    one.name ++ ' fights ' ++ two.name ++ '. ' ++ two.name ++ ' wins!'
  );
```

然后，我们打算让 `MinorVampire` 们互相争斗，看看会得到怎样的结果。我们有四个 `MinorVampire`（三个无名的女吸血鬼加上露西）。这里我们现将所有小鬼的力量值设置为 9：

```edgeql
update MinorVampire filter not exists .strength
set {
  strength := 9
};
```

现在，让我们将她们（`MinorVampire`）输入函数，试想一下输出会是什么？

```edgeql
select fight_2(MinorVampire, MinorVampire);
```

输出是……

……

```
{
  'Lucy fights Lucy. Lucy wins!',
  'Vampire Woman 1 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 2 fights Vampire Woman 2. Vampire Woman 2 wins!',
  'Vampire Woman 3 fights Vampire Woman 3. Vampire Woman 3 wins!',
}
```

该函数只被使用了四次，因为每次只有一个对象的集合进入到函数……每个 `MinorVampire` 都在和她自己战斗。这不是我们想要的。现在让我们用本地类型的别名再来尝试一下：

```edgeql
with M := MinorVampire,
select fight_2(M, MinorVampire);
```

顺便说一句，这实际上与下面这句完全相同：

```edgeql
select fight_2(MinorVampire, detached MinorVampire);
```

现在，输出变多了：

```
{
  'Lucy fights Lucy. Lucy wins!',
  'Lucy fights Vampire Woman 1. Lucy wins!',
  'Lucy fights Vampire Woman 2. Lucy wins!',
  'Lucy fights Vampire Woman 3. Lucy wins!',
  'Vampire Woman 1 fights Lucy. Lucy wins!',
  'Vampire Woman 1 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 1 fights Vampire Woman 2. Vampire Woman 2 wins!',
  'Vampire Woman 1 fights Vampire Woman 3. Vampire Woman 3 wins!',
  'Vampire Woman 2 fights Lucy. Lucy wins!',
  'Vampire Woman 2 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 2 fights Vampire Woman 2. Vampire Woman 2 wins!',
  'Vampire Woman 2 fights Vampire Woman 3. Vampire Woman 3 wins!',
  'Vampire Woman 3 fights Lucy. Lucy wins!',
  'Vampire Woman 3 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 3 fights Vampire Woman 2. Vampire Woman 2 wins!',
  'Vampire Woman 3 fights Vampire Woman 3. Vampire Woman 3 wins!',
}
```

我们成功地让每个 `MinorVampire` 都与其他 `MinorVampire` 进行了战斗，但仍然有 `MinorVampire` 和自己战斗（露西对露西，女人 1 对女人 1，等等）的情况。该问题的解决，正是体现本地类型别名的便利之处：例如，我们可以对其进行过滤，只允许彼此不同的对象使用 `fight_2()`：

```edgeql
with M := MinorVampire,
select fight_2(M, MinorVampire) filter M != MinorVampire;
```

现在我们终于让每个 `MinorVampire` 都与其他 `MinorVampire` 进行了战斗，且没有重复。

```
{
  'Lucy fights Vampire Woman 1. Lucy wins!',
  'Lucy fights Vampire Woman 2. Lucy wins!',
  'Lucy fights Vampire Woman 3. Lucy wins!',
  'Vampire Woman 1 fights Lucy. Lucy wins!',
  'Vampire Woman 1 fights Vampire Woman 2. Vampire Woman 2 wins!',
  'Vampire Woman 1 fights Vampire Woman 3. Vampire Woman 3 wins!',
  'Vampire Woman 2 fights Lucy. Lucy wins!',
  'Vampire Woman 2 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 2 fights Vampire Woman 3. Vampire Woman 3 wins!',
  'Vampire Woman 3 fights Lucy. Lucy wins!',
  'Vampire Woman 3 fights Vampire Woman 1. Vampire Woman 1 wins!',
  'Vampire Woman 3 fights Vampire Woman 2. Vampire Woman 2 wins!',
}
```

完美！

注意：这里仅使用 `detached` 是行不通的：`select Fight_2(MinorVampire, detached MinorVampire) filter MinorVampire != detached MinorVampire;` 是解决不了问题的，因为第一个 `detached MinorVampire` 不是变量名。如果没有一个名称用来访问，下一个 `detached MinorVampire` 只会是一个新的 `detached MinorVampire`，与前一个没有关系。

那么我们是否可以像创建 `CrewmanInBulgaria` 别名时所做的一样，用 `with` 同样做到创建一个别名并未其添加所需类型的链接和属性呢？答案是可以的。我们可以使用 `select` 并在其后面的 `{}` 中添加任何你想要的新链接和属性来做到这一点。下面是一个简单的例子：

```edgeql
with NPCExtraInfo := (
    select NPC {
      would_win_against_dracula := .strength > Vampire.strength
    }
  )
select NPCExtraInfo {
  name,
  would_win_against_dracula
};
```

结果如下，看起来没有人能赢德古拉：

```
{
  default::NPC {name: 'Jonathan Harker', would_win_against_dracula: {false}},
  default::NPC {name: 'The innkeeper', would_win_against_dracula: {false}},
  default::NPC {name: 'Mina Murray', would_win_against_dracula: {false}},
  default::NPC {name: 'John Seward', would_win_against_dracula: {false}},
  default::NPC {name: 'Quincey Morris', would_win_against_dracula: {false}},
  default::NPC {name: 'Arthur Holmwood', would_win_against_dracula: {false}},
  default::NPC {name: 'Abraham Van Helsing', would_win_against_dracula: {false}},
  default::NPC {name: 'Lucy Westenra', would_win_against_dracula: {false}},
  default::NPC {name: 'Renfield', would_win_against_dracula: {false}},
}
```

现在假设德古拉已经实现了他的所有目标，统治了伦敦。我们将为其创建一个叫做 `DraculaKingOfLondon` 的新的快速类型（别名），包含一个更完善的名字以及一个指向 `subjects`（= 被统治的人）的链接，该链接指向所有去过伦敦的 `Person`。然后我们来选择（查询）这个新类型，并计算有多少个 `subjects` 已被德古拉统治。具体操作如下所示：

```edgeql
with DraculaKingOfLondon := (
    select Vampire {
      name := .name ++ ', King of London',
      subjects := (select Person filter 'London' in .places_visited.name),
    }
  )
select DraculaKingOfLondon {
  name,
  subjects: {name},
  number_of_subjects := count(.subjects)
};
```

输出为：

```
{
  default::Vampire {
    name: 'Count Dracula, King of London',
    subjects: {
      default::NPC {name: 'Jonathan Harker'},
      default::NPC {name: 'The innkeeper'},
      default::NPC {name: 'Mina Murray'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
      default::NPC {name: 'Abraham Van Helsing'},
      default::NPC {name: 'Lucy Westenra'},
      default::NPC {name: 'Renfield'},
    },
    number_of_subjects: 9,
  },
}
```

[→ 点击这里查看到第 17 章为止的所有代码](code.md)

<!-- quiz-start -->

##小测验

1. 试试只用单行代码显示所有 NPC 的姓名、力量值、其到访城市的名称和该城人口，以及 NPC 年龄（如果年龄 = `{}`，则显示 0）。

2. 上题的查询结果中显示了很多没有上下文的数字，如何能使其变得更友好易读呢？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森侦探。_
