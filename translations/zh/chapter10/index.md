---
tags: Tuples, Computed Properties, Math
---

# 第十章 - 惠特比的可怕事件

> 米娜（Mina）和露西（Lucy）现在正在惠特比（Whitby）享受着她们愉快的时光。一天晚上，大风暴突然来袭，一艘船在雾中缓缓靠岸——这正是德古拉所乘坐的德米特号（the Demeter）。此后，露西开始在晚上梦游，她脸色苍白，一直说些奇怪的话。米娜试图阻止她，但有时露西会跑到外面去。一天晚上，露西看着快要下山的太阳说道：“他的眼睛又红了！他们是一样的。”米娜很担心，于是她向苏厄德医生（Dr. Seward）寻求帮助。苏厄德医生对露西做了检查，发现她十分虚弱，但他并不知道这是为什么。苏厄德医生决定打电话向来自荷兰的老师亚伯拉罕·范海辛 (Abraham Van Helsing) 寻求帮助。范海辛检查露西后，显得很震惊。然后他转向其他人说道：“听着。我们可以帮助这个女孩，但方法可能会令你们感到很奇怪。你们必须相信我……”

惠特比市（Whitby）位于英格兰（England）东北部。现在我们的 `City` 类型只是扩展了 `Place`，`Place` 只有属性 `name`、`modern_name` 和 `important_places`。现在可能是增加 `property population`（人口属性）的好时机，它可以帮助我们在游戏中具象一个城市。我们将使用 `int64` 来表达人口的数量：

```sdl
type City extending Place {
  population: int64;
}
```

顺便说一下，这是小说出版时相关各城市的大致人口。这些城市的人口在 1893 年时还很少：

- Buda-Pesth (Budapest): 402706
- London: 3500000
- Munich: 230023
- Whitby: 14400
- Bistritz (Bistrița): 9100

现在让我们来插入惠特比市（Whitby），这很容易：

```edgeql
insert City {
  name := 'Whitby',
  population := 14400,
  important_places := ['Whitby Abbey']
};
```

但是对于其他已经在数据库中存在城市来说，我们需要同时做一下关于人口数量的更新。

## 元组和数组的使用

如果我们将所有城市数据放在一起，我们完全可以使用 `for` 和 `union` 循环进行一次“单次”插入（或更新）。假设我们在一个“元组（tuples）”中存放一些数据，它看起来与数组（arrays）相似，但又完全不同。一个很大的区别是元组可以包含不同类型的数据，比如像下面这样：

`('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)`

在这种情况下，该类型称为 `tuple<str, int64>`。

在我们开始使用这些元组之前，让我们先来了解一下元组（tuples）与数组（arrays）之间的区别。首先，让我们先进一步看一下切片数组（slicing arrays）和字符串（strings）。

你一定还记得我们使用方括号来访问数组或字符串的一部分。所以 `select ['Mina Murray', 'Lucy Westenra'][1];` 将给出输出 `{'Lucy Westenra'}`（即索引 1 对应的数值）。

你也应该还记得，我们在索取切片时用冒号分隔起始索引和结束索引，如下所示：

```edgeql
select NPC.name[0:10];
```

这将输出所有 NPC 名字的前十个字母：

```
{
  'The innkee',
  'Mina Murra',
  'Jonathan H',
  'Lucy Weste',
  'John Sewar',
  'Quincey Mo',
  'Arthur Hol',
  'Renfield',
}
```

如果你想从后往前（倒着）进行索引，同样可以用负数来完成。例如：

```edgeql
select NPC.name[2:-2];
```

这将从索引 2 打印到距离末尾 2 个索引（即分别切断名字前后两端的两个字母）的所有字母。输出如下所示：

```
{
  'e innkeep',
  'na Murr',
  'nathan Hark',
  'cy Westen',
  'hn Sewa',
  'incey Morr',
  'thur Holmwo',
  'nfie',
}
```

元组完全不同：它们的行为更像一个对象类型，只是其属性的名称由数字进行了替代。这就是元组可以将不同类型的数据存储在一起的原因：`string` 和 `array`，`int64` 和 `float32`，等等。

所以像下面这样，完全没有问题：

```edgeql
select {('Bistritz', 9100, cal::to_local_date(1893, 5, 6)), ('Munich', 230023, cal::to_local_date(1893, 5, 8))};
```

输出结果是：

```
{
  ('Bistritz', 9100, <cal::local_date>'1893-05-06'),
  ('Munich', 230023, <cal::local_date>'1893-05-08'),
}
```

由于类型已经设定（上面所示的类型是 `tuple<str, int64, cal::local_date>`），因此你不能将它与其他元组类型混淆。比如像下面这样是不被允许的：

```edgeql
select {(1, 2, 3), (4, 5, '6')};
```

此时，EdgeDB 会报错，因为这里它不会尝试处理不同类型的元组。错误提示为：

```
ERROR: QueryError: operator 'union' cannot be applied to operands of type 'tuple<std::int64, std::int64, std::int64>' and 'tuple<std::int64, std::int64, std::str>'
  Hint: Consider using an explicit type cast or a conversion function.
```

在上面的例子中，我们可以轻松地将第二个元祖中的最后一个字符串转换为整数，然后 EdgeDB 则会再次工作，即：`select {(1, 2, 3), (4, 5, <int64>'6')};`。

要访问元组的字段，仍然是从数字 0 开始索引，但数字要写在 `.` 之后，而不是在 `[]` 内。现在我们已经足够了解元祖了，于是我们可以像下面这样同时对所有城市的人口数量进行一次“单次“更新了：

```edgeql
for data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
union (
  update City filter .name = data.0
  set {
    population := data.1
  }
);
```

即，将元组依次发送到 `for` 循环中，按城市名字的字符串（即 `data.0`）进行过滤，然后对城市人口数量（即 `data.1`）进行更新。

在这个部分的最后，我们再说一下关于类型的转换。我们知道我们可以用 `<>` 在标量类型间做转换，这也同样适用于标量类型的元组，且会使用与 `<>` 相同的格式，只是需要把它放在 `<tuple>` 里面，像这样：

```edgeql
with london := ('London', 3500000),
select <tuple<json, int32>>london;
```

输出为：

```
{("\"London\"", 3500000)}
```

下面是另一个例子，假设我们需要对伦敦人口进行一些浮点数的数学计算：

```edgeql
with london := <tuple<json, float64>>('London', 3500000),
  select (london.0, london.1 / 9);
```

输出为：`{("\"London\"", 388888.8888888889)}`.

## 排序和数学

现在我们有了一些数字，因此我们可以开始玩排序和数学了。排序是非常简单的：输入 `order by`，然后指明你要排序的属性或链接。这里我们按人口排序：

```edgeql
select City {
  name,
  population
} order by .population desc;
```

输出结果是：

```
{
  default::City {name: 'London', population: 3500000},
  default::City {name: 'Buda-Pesth', population: 402706},
  default::City {name: 'Munich', population: 230023},
  default::City {name: 'Whitby', population: 14400},
  default::City {name: 'Bistritz', population: 9100},
}
```

什么是 `desc`？它是指降序，因此这里先展示人口数量最大的，然后逐步减小。如果我们不写 `desc`，那么 EdgeDB 会假设我们需要的是升序。你也可以特意写上表达升序的 `asc`（使代码阅读起来更清晰），但你不是非得这么做。

对于一些实用的数学函数，你可以查看 `std` 中的 {eql:func}`函数 <docs:std::sum>` 以及 `math` {ref}`模块 <docs:ref_std_math>`。不过，与其逐一查看，不如让我们这在里一起做一个大查询，将其中的一些函数一并运用进来。为了使输出更友好，我们会对各函数作用后的结果进行解释，并编写进字符串，同时将函数的输出结果转换为 `<str>`，最后，使用级联运算符 `++` 将它们连接在一起。如下所示：

```edgeql
with cities := City.population
select (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>math::mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>math::stddev(cities)
);
```

这里使用了相当多的函数：

- `count()` 计算项目（item）的数量，
- `all()` 如果所有项目都匹配，则返回 `{true}`，否则返回 `{false}`，
- `sum()` 对所有项目进行相加，
- `max()` 给出数值最大的项目，
- `min()` 给出数值最小的项目，
- `math::mean()` 给出所有项目的平均值，
- `any()` 只要有一个项目匹配，则返回 `{true}`，否则返回 `{false}`，
- `math::stddev()` 给出所有项目的标准差。

输出结果也清楚地说明了它们是如何工作的：

```
{
  (
    'Number of cities: 5',
    'All cities have more than 50,000 people: false',
    'Total population: 4156229',
    'Smallest and largest population: 9100, 3500000',
    'Average population: 831245.8',
    'At least one city has more than 5 million people: false',
    'Standard deviation: 1500876.8248',
  ),
}
```

其中 `any()`、`all()` 和 `count()` 在操作中特别有用，它们可以让你更加了解你的数据。

## 使用 with 导入模块

你也可以使用关键字 `with` 来导入模块。在上面的例子中，我们使用了 EdgeDB `math` 模块中的两个函数：`math::mean()` 和 `math::stddev()`。如果我们只写 `mean()` 和 `stddev()`，则会报错：

```
ERROR: InvalidReferenceError: function 'default::mean' does not exist
```

如果你不想每次都写模块名 `math`，你可以在 `with` 后面导入模块。让我们来调整一下上面提到的大查询语句，看看你是否发现了变化：

```edgeql
with cities := City.population,
  module math
select (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>stddev(cities)
);
```

输出结果是一样的，但我们添加了一个 `math` 模块的导入，使得我们在后面的语句中只需要写 `mean()` 和 `stddev()`。

与“重命名一个类型”相同，你还可以使用 `as` 重命名一个模块（其实就是为模块起个 _别名_）。所以像下面这样也可以正常工作：

```edgeql
with M as module math,
select M::mean(City.population);
```

结果是所有城市人口数量的平均值：`{831245.8}`。

## 更多的称呼属性

我们在本章中看到苏厄德医生（Dr. Seward）请他的老师范海辛医生 (Dr. Van Helsing) 来帮助露西（Lucy）。以下是范海辛医生回信中的开头：

```
Letter, Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc., to Dr. Seward.

“2 September.

“My good Friend,—
“When I have received your letter I am already coming to you.
```

`Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc.` 这部分很有趣。这可能是个不错的时机让我们进一步考虑 `Person` 类型中的 `name` 属性。现在我们的 `name` 属性只是一个字符串，但是我们的角色会拥有不同类别的称呼，并按以下顺序进行排列：

Title（头衔） | First name（名） | Last name（姓） | Degree（学位）

所以小说里的称呼有 'Count Dracula'（德古拉伯爵，头衔和姓），'Dr. Seward'（医生西沃德，头衔和姓），'Dr. Abraham Van Helsing, M.D, Ph. D. Lit.'（医生亚伯拉罕·范海辛医学博士、哲学博士、文学博士等，头衔+名字+姓氏+学位），依此类推。

这会让我们认为我们应该有类似 `first_name`、`last_name`、`title` 等称呼的属性，然后使用计算（computed）属性将它们连接在一起展示全称。但话又说回来，并不是每个角色的名字都有这四个部分，比如：“女人 1（Woman 1）”和“旅店老板（The Innkeeper）”，我们的游戏中更多会是这样的命名。因此，直接去掉 `name` 或总是组合不同的部分构建名称可能不是一个好主意。但是在我们的游戏中，我们又会让角色写信或互相交谈，他们将不得不使用诸如头衔和学位之类的称呼。

我们可以尝试采用中间方法。保留 `name`，并为 `Person` 添加一些可选属性：

```sdl
title: str;
degrees: str;
property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
```

我们也可以尝试对 `degrees` 做一些更有趣的事情，比如将 `degrees` 设置为 `array<str>`，但我们的游戏可能不需要那么高的精度，我们只是会在角色对话框中引用到他们的学位称号而已。

现在，让我们来插入创建范海辛 (Van Helsing）：

```edgeql
insert NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```

接下来，我们可以利用这些属性在游戏中模仿一些对话了。例如：

```edgeql
with helsing := (select NPC filter .name ilike '%helsing%')
select (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ ',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

顺便说一下，字符串中的 `\n` 用来创建新行，而 `\t` 用于将文本向右移动一个制表符。

我们得到结果：

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.,
        I am sorry to say that I bring bad news about Lucy.',
  ),
}
```

PS：在有真实”用户“的标准数据库中，收集用户信息要简单得多：因为我们可以让用户自己输入他们的名字、姓氏等，并使每个部分都成为一个属性。

## 转义字符及原始字符串

除了 `\n` 和 `\t` 之外，还有很多其他转义字符——你可以在 {ref}`这里 <docs:ref_eql_lexical_str_escapes>` 查看完整列表。有些很少见，但带有 `\x` 的十六进制可能是一个很有用的例子。

如果你想忽略转义字符，请在引号前放置一个 `r`。让我们用上面的例子来试试，如下所示，我们在最后一部分的引号前放置了一个 `r`：

```edgeql
with helsing := (select NPC filter .name ilike '%helsing%')
select (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ r',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

于是我们会得到：

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.,\\n\\tI am sorry to say that I bring bad news about Lucy.',
  ),
}
```

最后，如果你想保有字符串完全原始的模样，你可以在其两侧使用 `$$`。里面内容将忽略所有的引号，因此你不必担心字符串会在中间断开。下面是一个包含了一堆单引号和双引号的示例：

```edgeql
select $$ 
"Dr. Van Helsing would like to tell "them" 
about "vampires" and how to "kill" them, 
but he'd sound crazy."
$$;
```

输出是：`{' "Dr. Van Helsing would like to tell "them" about "vampires" and how to "kill" them, but he\'d sound crazy." '}`。如果没有 `$$`，看起来像是四个独立的字符串被三个未知的关键字所连接，并且会产生错误。

## 所有标量类型

你现在已经了解了 EdgeDB 的所有标量类型。总结起来有：`int16`、`int32`、`int64`、`float32`、`float64`、`bigint`、`decimal`、`sequence`、`str`、`bool`、`datetime`、 `duration`、`cal::local_datetime`、`cal::local_date`、`cal::local_time`、`cal::relative_duration`, `cal::date_duration`, `uuid`、`json` 和 `enum`。你可以在 {ref}`这里 <docs:ref_datamodel_scalar_types>` 查看它们的文档。

## 关键词 unless conflict on + else + update

之前，我们在 `Person` 的 `name` 上设置了 `constraint exclusive`，这样游戏中就不会出现两个同名的角色了。之所以这样做，是因为我们担心有人看到了书中的一个角色并将其插入，然后其他人（协作的人）又尝试做同样的事情。因此，下面这个名为约翰尼（Johnny）的角色可以被成功插入，说明我们的数据库里暂时还没有同名的对象：

```edgeql
insert NPC {
  name := 'Johnny'
};
```

如果我们再重复做一遍上面的插入，我们则会得到错误提示：`ERROR: ConstraintViolationError: name violates exclusivity constraint`

但有时仅仅生成错误提示是不够的——也许我们希望在错误发生时会引发其他的事情而不是直接放弃。这正是 `unless conflict on` 发挥作用的时候了，在它的后面会跟着 `else` 来说明发生错误时具体要做什么。通过示例可能更容易解释 `unless conflict on`。下面展示了当我们要创建带有人口的 `City` 时我们可以怎么做，即：插入，但如果该城市已经存在于数据库中，则做更新。

1880 年慕尼黑（Munich）的人口为 230,023，五年后为 261,023。假设我们正在更新 `City` 数据，且一些城市可能已经在数据库里存在了，而另一些城市可能尚未创建。那么，`insert` 慕尼黑（Munich）的语句应如下所示：

```edgeql
insert City {
  name := 'Munich',
  population := 261023,
} unless conflict on .name
else (
  update City
  set {
    population := 261023,
  }
);
```

让我们一步一步地拆解：

首先是一个普通的插入：

```edgeql
insert City {
  name := 'Munich'
}
```

但是数据库里可能已经有一个同名的 `City` 了，所以它可能不起作用，于是我们添加 `unless conflict on .name`。然后在它后面加上 `else`，指明如果该城市已经存在，该做些什么。

需要注意的是：这里我们不写 `else insert`，因为我们已经在 `insert` 当中了。我们是要为插入类型在插入失败的情况下做 `update`，所以是 `update City`。根本上讲，我们是从插入中取出失败的数据并更新它以重试。所以我们会这样做：

```edgeql
else (
  update City
  set {
    population := 261023
  }
);
```

这样做，我们可以确保得到一个名为慕尼黑（Munich）且人口为 261,023 的 `City` 对象，无论它是否已经存在于数据库中。

[→ 点击这里查看到第 10 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 尝试通过一个插入语句插入两个 `NPC` 类型的对象，其中包含 `name`, `first_appearance` 和 `last_appearance` 信息。

   `{('Jimmy the Bartender', '1893-09-10', '1893-09-11'), ('Some friend of Jonathan Harker', '1893-07-08', '1893-07-09')}`

2. 这里还有两个要插入的 `NPC`，后者的最后有一个空集（因为她还没有死）。插入它们时，我们会遇到什么问题？

   `{('Dracula\'s Castle visitor', '1893-09-10', '1893-09-11'), ('Old lady from Bistritz', '1893-05-08', {})}`

3. 你将如何按人物姓名的最后一个字母对 `Person` 类型的对象进行排序？

4. 尝试插入一个名为 `''` 的 `NPC`。现在，你将如何在问题 3 中执行相同的查询？

   提示：`''` 的长度为 0，这可能会有问题。

5. 范海辛医生（Dr. Van Helsing ）有一份 `MinorVampire` 的名单，上面有他们的名字和力量。我们的数据库中已经有一些 `MinorVampire` 了。如果对象已经存在，你将如何在确保 `update` 的同时 `insert` 不存在的？ 


[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _他们会相信范海辛所说的真相吗？_
