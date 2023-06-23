---
tags: Writing Functions, Multiplication, Coalescing
---

# 第十一章 - 露西怎么了？

> 范海辛医生（Dr. Van Helsing）认为露西（Lucy）正在被吸血鬼纠缠，但他还没有告诉其他人，因为他觉得没有人会相信的。于是，他告诉大家需要关上窗户，并在所有地方都放上大蒜。大家很困惑，但苏厄德医生（Dr. Seward）让他们照办，因为范海辛医生是他认识的人力最有学识的。见效了！露西好起来了。但一天晚上，露西的母亲走进房间后，觉得屋里闻起来太糟糕了！于是为了换换空气，她打开了窗户。第二天，露西醒来，脸色苍白，再次陷入病重。每次有人犯这样的错误，德古拉（Dracula）都会潜入露西的房间使其变得虚弱，而每次病重时，露西都需要男士们献血给她才能好起来。与此同时，伦菲尔德（Renfield）还在试图吃活物，苏厄德医生（Dr. Seward）无法理解他。突然有一天，伦菲尔德不想再多费口舌，只是说了句：“我不想和你说话，你现在不配；主人就在我身边。” 

可以看出，越来越多的多角色参与事件出现在了本书中。有些事件是（爱慕露西的）三个男人和范海辛博士（Dr. Van Helsing）一起参与的，有些事件只有露西（Lucy）和德古拉（Dracula）。之前还有事件发生在乔纳森·哈克（Jonathan Harker）和德古拉（Dracula）之间，还有乔纳森·哈克和三个女吸血鬼之间，等等。在我们的游戏中，我们可以使用 `Event` 类型将所有内容组合在一起，如：人物、时间、地点等等。

这个 `Event` 类型的定义有点长，但它会是用来表达游戏事件的主要类型，所以这里需要进行详细的说明。我们可以这样定义：

```sdl
type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  location: tuple<float64, float64>;
  east: bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.location.0 ++ '_N_' ++ <str>.location.1 ++ '_' ++ ('E' if .east else 'W');
}
```

你可以看到大多数属性或链接都有 `required`，因为如果 `Event` 类型没有我们需要的所有信息，它就没有什么用处了。它总是需要描述、时间、地点和参与人员。有趣的部分是 `url` 属性：它是一个计算（computed）属性，如果我们需要，它可以为我们提供地图中指向事件位置的确切 url。

我们生成的 url 需要知道这个位置是在格林威治（Greenwich）的东部还是西部，以及它们是北部还是南部。如下是 Bistritz 的 url：

`https://geohack.toolforge.org/geohack.php?pagename=Bistri%C8%9Ba&params=47_8_N_24_30_E`

对我们来说，幸运的是书中的事件都发生在地球的北部。所以 `N` 总是会在那里。但有时他们在格林威治东部，有时在西部。为了说明是东方还是西方，我们可以使用一个简单的 `bool`。然后在 `url` 属性中，我们将所有相关属性放在一起以创建链接，如果 `east` 为 `true`，则以 `'E'` 结束，否则以 `'W'` 结束。

（当然，如果我们接收的经度是简单的正负数（+ 表示东，- 表示西），那么 `east` 也可以是一个计算（computed）属性：`property east := true if location.0 > 0 else false`。但是对于我们的架构，我们会假设我们可以从某个地方以类似 `[50.6, 70.1, true]` 的这种格式获取到位置信息，然后直接赋予 `location` 和 `east`）

现在，让我们来插入本章中的一个事件。它发生在 9 月 11 日晚上，当时范海辛医生（Dr. Van Helsing）正试图帮助露西（Lucy）。你可以看到 `description` 属性只是我们编写的一个字符串，以便于之后进行搜索。它可长可短，这取决于你，我们甚至可以把书中的某些部分粘贴进去。

```edgeql
insert Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1893, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1893, 9, 11, 23, 0, 0),
  place := (select Place filter .name = 'Whitby'),
  people := (select Person filter .name ilike {'%helsing%', '%westenra%', '%seward%'}),
  location := (54.4858, 0.6206),
  east := false
};
```

有了所有这些信息，我们则可以通过描述、角色、位置等信息来查询事件。

现在让我们查询所有描述中包含 `garlic flowers` 一词的事件：

```edgeql
select Event {
  description,
  start_time,
  end_time,
  place: {
    name
  },
  people: {
    name
  },
  location,
  url
} filter .description ilike '%garlic flowers%';
```

这生成了一个不错的输出，向我们展示了结果事件的所有信息： 

```
{
  default::Event {
    description: 'Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.',
    start_time: <cal::local_datetime>'1893-09-11T18:00:00',
    end_time: <cal::local_datetime>'1893-09-11T23:00:00',
    place: {default::City {name: 'Whitby'}},
    people: {
      default::NPC {name: 'Lucy Westenra'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Abraham Van Helsing'},
    },
    location: (54.4858, 0.6206),
    url: 'https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W',
  },
}
```

url 也是正确的。它是：<https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W> 点击它可以直接定位到惠特比市。

## 自定义函数

在之前的章节中，我们给伦菲尔德（Renfield）的力量值设为了 10，和力量仅是 5 的乔纳森（Jonathan）相比，我们可以看出他非常的强壮。

现在，我们用它来实践“如何制作函数”。由于 EdgeQL 是强类型的，因此你必须在签名中同时指明输入类型和返回类型。例如，输入 int16 类型并返回 float64 类型的函数签名如下所示：

```sdl
function does_something(input: int16) -> float64
```

`->` 细箭头用于指明返回值。

对于函数体的编写，我们按照下方步骤进行：

- 写下 `using` 然后跟上一个 `()` 括号，
- 在括号里写下函数定义，
- 用分号结束定义（注意：是在 `)` 后面写分号）。 

下面是一个非常简单的函数，它接受一个数字并最终返回一个字符串：

```sdl
function make_string(input: int64) -> str
  using (<str>input);
```

就是这样！

Let's write a quick function to make our Event type a little nicer to read. Instead of putting `'https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W'` inside the `Event` type, we can make a function called `get_url()` that simply returns this `str` for us. With that, our `url` property definition is 42 characters shorter. Let's add this function to the schema and change the `url` in the `Event` type to use it:

```sdl
function get_url() -> str
  using (<str>'https://geohack.toolforge.org/geohack.php?params=');

type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  location: tuple<float64, float64>;
  east: bool;
  property url := get_url() ++ <str>.location.0 ++ '_N_' ++ <str>.location.1
  ++ '_' ++ ('E' if .east else 'W');
}
```

现在让我们来编写一个函数用于两个角色之间的战斗。我们将使逻辑尽可能简单：即具有更多力量的角色获胜，如果他们的力量相同，则第二个玩家获胜。

```sdl
function fight(one: Person, two: Person) -> str
  using (
    one.name ++ ' wins!'
    if one.strength > two.strength
    else two.name ++ ' wins!'
  );
```

这个函数“看起来”还不错，但是当你尝试创建它时，你会得到这样的错误提示：

```
InvalidFunctionDefinitionError: return cardinality mismatch in function
  declared to return exactly one value
```

之所以会出现这个错误，是因为 `name` 和 `strength` 在 `Person` 类型中不是必须的（没有标记 required）。如果你传入的是一个没有 `name` 值或没有 `strength` 值的 `Person` 类型，这个函数将会返回空集。（对此，我们会在下一个章节有更进一步的说明。）这并不是 EdgeDB 想要的，因为我们在函数定义时告诉了 EdgeDB：这个函数会返回一个字符串。

当然，我们可以返回去，在 `Person` 类型中在 `name` 和 `strength` 前加上 `required`，但我们则需要确保所有已经存在的 `Person` 对象在他们的 `name` 和 `strength` 上都有了相应的数值。可以想见，这会给我们带来很多麻烦，因此，这里我们也并不打算这样去解决这个问题。

## 用合并操作符提供备选方案（fallback）

解决上面所遇问题最简单的方法则是为可能未设置的属性提供某种备选方案（fallback）。比如：如果 `one` 没有名字，我们可以将它们称为 `Fighter 1`。如果某人没有力量值，我们可以将他们的力量默认为 `0`。

为此，我们可以使用合并操作符 {eql:op}`coalescing operator <docs:coalesce>`，即 `??`。如果操作符左边不为空，则结果就是左边的内容。否则，结果会使用操作符右边的内容。

下面是一个简单的例子：

```
db> select <str>{} ?? 'Count Dracula is now in Whitby';
```

`??` 的左边是空集，因为它 _是_ 空集，所以合并运算符在这里将放弃使用它，转而去查看右边的内容，因此，这个查询结果将使用合并运算符右侧生成的字符串：`{'Count Dracula is now in Whitby'}`

如果运算符 `??` 两边都不是空集，则合并运算符将使用运算符左边的内容进行返回。如果两边都是空集，则会返回空集。

下面就让我们使用合并运算符来修复我们在上面 `fight()` 函数中遇到的问题：

```sdl
function fight(one: Person, two: Person) -> str
  using (
    (one.name ?? 'Fighter 1') ++ ' wins!'
    if (one.strength ?? 0) > (two.strength ?? 0)
    else (two.name ?? 'Fighter 2') ++ ' wins!'
  );
```

现在，EdgeDB 对于那些返回值可能是空集的属性提供了备选内容，即：如果 `one.name` 是空集，我们则会使用 `'Fighter 1'`。如果其中一个人的 `strength` 属性是空集，我们则会使用 `0`。同样，如果 `two.name` 是空集，我们则会使用 `'Fighter 2`。这将确保函数始终可以返回我们承诺的字符串响应。

到目前为止，只有乔纳森（Jonathan）和伦菲尔德（Renfield）拥有 `strength` 属性，所以现在我们来让他们在这个新的 `fight()` 函数中较量一番：

```edgeql
with
  renfield := (select Person filter .name = 'Renfield'),
  jonathan := (select Person filter .name = 'Jonathan Harker')
select (
  fight(jonathan, renfield)
);
```

结果符合我们的预期：`{'Renfield wins!'}`

当我们通过过滤器选择人物时，我们最好加上 `assert_single()`。因为 EdgeDB 返回的是集合，如果它得到多个结果，那么 EdgeDB 将针对每个可能的组合都使用一次该函数。EdgeDB 处理这种情况的方法叫做“笛卡尔乘法”（Cartesian multiplication），现在让我们来进一步了解一下。

## 笛卡尔乘法

笛卡尔乘法（Cartesian multiplication）听起来很吓人，但实际上只是意味着“将一个集合中的每个项目分别连接（join）到另一个集合中的每个项目”。通过图示可能会更加容易理解，幸运的是维基百科已经为我们制作了插图。当你在 EdgeDB 中将两个集合相乘时，你会得到笛卡尔乘积，如下所示：

![](cartesian_product.svg)

来源：[维基百科的用户“quartl”](https://en.wikipedia.org/wiki/Cartesian_product#/media/File:Cartesian_Product_qtl1.svg)

这意味着如果我们对 `Person` 进行 `select` 后并传给函数 `fight()`，EdgeDB 将按照下面的公式运行函数：

- `{the number of items in the first set}` \* `{the number of items in the second set}`

因此，如果第一个集合中有两个，第二个结合中有三个，该函数将被运行 6 次。

为了演示，我们将给函数的两个输入都传入三个对象，以此来测试我们的 `flight()` 函数。这里，对于那些还没有力量值的其他角色，我们暂时将其力量值均设置为 5：

```edgeql
update Person filter not exists .strength
set {
  strength := 5
};
```

此外，我们还将运用 `++` 使得输出结果更加清晰可读：

```edgeql
with
  first_group := (select Person filter .name in {'Jonathan Harker', 'Count Dracula', 'Arthur Holmwood'}),
  second_group := (select Person filter .name in {'Renfield', 'Mina Murray', 'The innkeeper'}),
select (
  first_group.name ++ ' fights against ' ++ second_group.name ++ '. ' ++ fight(first_group, second_group)
);
```

输出如下所示：总共有九场战斗，第 1 组中的每个人与第 2 组中的每个人分别战斗一次。

```
{
  'Jonathan Harker fights against Renfield. Renfield wins!',
  'Jonathan Harker fights against The innkeeper. The innkeeper wins!',
  'Jonathan Harker fights against Mina Murray. Mina Murray wins!',
  'Arthur Holmwood fights against Renfield. Renfield wins!',
  'Arthur Holmwood fights against The innkeeper. The innkeeper wins!',
  'Arthur Holmwood fights against Mina Murray. Mina Murray wins!',
  'Count Dracula fights against Renfield. Renfield wins!',
  'Count Dracula fights against The innkeeper. The innkeeper wins!',
  'Count Dracula fights against Mina Murray. Mina Murray wins!',
}
```

如果你去掉过滤器，只用 `select Person`，你会得到超过 100 个结果。而 EdgeDB 默认只显示前 100 个，在显示 100 个结果后会显示：

`` ... (further results hidden `\set limit 100`)``

[→ 点击这里查看到第 11 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测试

1. 编写一个名为 `get_lucies()` 的函数，使其只返回所有名称与“Lucy Westenra”匹配的 `Person` 对象？

2. 编写一个函数：接收两个字符串，并返回所有名称可以匹配输入的两个字符串中任意一个的 `Person` 对象？

   提示：尝试使用 `set of Person` 作为返回类型。

3. 下面的语句将会输出什么？

   ```edgeql
   select {'Jonathan', 'Arthur'} ++ {' loves '} ++ {'Mina', 'Lucy'} ++ {' but '} ++ {'Dracula', 'The inkeeper'} ++ {' doesn\'t love '} ++ {'Mina', 'Jonathan'};
   ```

4. 如何制作一个函数来计算一个城市比另一个城市大多少倍？

5. `select (City.population + City.population)` 和 `select ((select City.population) + (select City.population))` 会产生不同的结果吗？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _一天晚上，露西：“是什么在拍打窗户？听起来像是蝙蝠什么的……"_
