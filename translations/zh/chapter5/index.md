---
tags: Datetime, Describing Types
leadImage: illustration_05.jpg
---

# 第五章 - 乔纳森试图离开城堡

可怜的乔纳森（Jonathan）真是倒霉。那么，在这一章里他又发生了什么呢？

> 白天，乔纳森决定尝试探索这座城堡，但太多的门窗都被锁上了。他不知道怎么出去，但希望至少能给米娜寄一封信。他依旧假装毫无疑问，在夜里继续不停地和德古拉聊天。一天晚上，他看到德古拉爬出窗口，像蛇一样爬下城墙。乔纳森现在感到十分害怕，他猜想德古拉大概不是人类。几天后，他打破了一扇门，发现了城堡的另一部分。这个房间很奇怪，让他控制不住犯困。当他睁开眼睛醒来时，他看到了三个吸血鬼女人站在他的身旁。乔纳森被她们吸引的同时又感到害怕。他想亲吻她们，但他知道如果他这样做了就死定了。她们靠得更近了，但乔纳森无法动弹……

## 日期时间（std::datetime）

由于乔纳森（Jonathan）想起了已经回到伦敦的未婚妻米娜（Mina），那么就让我们来了解一下带有时区的日期时间 `std::datetime`。要创建日期时间，你只需使用 `<datetime>` 转换一个符合 ISO 8601 格式的字符串。该格式如下所示：

`YYYY-MM-DDTHH:MM:SSZ`

实际日期看起来像这样：

`'2020-12-06T22:12:10Z'`

其中 `T` 只是一个分隔符，最后的 `Z` 代表“零时间线”（zero timeline）。这意味着它与 UTC 有 0° 的偏移：换句话说，它 _就是_ UTC。

获取 `datetime` 的另一种方法是使用 `to_datetime()` 函数。{eql:func}`这里 <docs:std::to_datetime>` 是它的函数签名，里面有六种方法可以通过使用此函数生成 `datetime`，具体使用哪个取决于你想要如何生成。EdgeDB 将根据你提供的输入得知你选择了六种方法中的哪一种。

顺便说一下，你可能会留意到一个不太熟悉的、名为 {eql:type}` ``decimal`` <docs:std::decimal>` 的类型。这是一个具有“任意精度”的浮点数，这意味着你可以根据需要在小数点后给出任意数量的数字。计算机上的浮点类型会由于舍入错误，在一段时间后 [变得不精确](https://www.youtube.com/watch?v=PZRI1IfStY0&ab_channel=Computerphile)。比如下面的例子：

```
db> select 6.777777777777777; # Good so far
{6.777777777777777}
db> select 6.7777777777777777; # Add one more digit...
{6.777777777777778} # Where did the 8 come from?!
```

如果你想避免这个问题，就在末尾添加一个 `n` 以获得一个 `decimal` 类型，它将尽可能做到精确。

```
db> select 6.7777777777777777n;
{6.7777777777777777n}
db> select 6.7777777777777777777777777777777777777777777777777n;
{6.7777777777777777777777777777777777777777777777777n}
```

同时，还有一个 `bigint` 类型，也可以使用 `n` 来表示任意大小。那是因为即使 int64 也有上限：它是 9223372036854775807。

```
db> select 9223372036854775807; # Good so far...
{9223372036854775807}
db> select 9223372036854775808; # But add 1 and it will fail
ERROR: NumericOutOfRangeError: std::int64 out of range
```

所以这里你可以加上一个 `n`，它将会创建一个可以容纳任何大小的 `bigint`。

```
db> select 9223372036854775808n;
{9223372036854775808n}
```

现在我们知道了所有的数字类型，让我们回到 `std::to_datetime` 函数的六个签名：

```
std::to_datetime(s: str, fmt: optional str = {}) -> datetime
std::to_datetime(local: cal::local_datetime, zone: str) -> datetime
std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime
std::to_datetime(epochseconds: decimal) -> datetime
std::to_datetime(epochseconds: float64) -> datetime
std::to_datetime(epochseconds: int64) -> datetime
```

如果你对 ISO 8601 不熟悉，或者你有一堆单独的数字要组成一个日期，那么最简单的方法可能是使用第三个。有了这个，我们可以对“时间”里的整数通过使用 `to_datetime()` 来获得其对应的正确的时间戳。

假设现在是 5 月 12 日（May 12）10:35，一个晴朗的早上，在德古拉城堡中。太阳升起了，德古拉在某个地方睡着，乔纳森正试图利用白天的时间逃出去给米娜寄一封信。在罗马尼亚，时区是“EEST”（东欧夏令时）。我们将使用 `to_datetime()` 来生成它。因为整个故事都发生在同一年，所以我们可以随便指定一个年份——为了方便，我们将使用 2020 年。于是我们输入：

```edgeql
select to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');
```

并得到以下输出：

`{<datetime>'2020-05-12T07:35:00Z'}`

`07:35:00` 部分说明时间已自动转换为 UTC，即米娜居住的伦敦。

我们还可以使用它来查看事件之间的持续时间。EdgeDB 有一个 `duration` 类型，你可以通过用一个日期时间减去另一个日期时间来获得。现在让我们练习一下，计算一个中欧日期和一个韩国日期之间相差的确切秒数：

```edgeql
select to_datetime(2020, 5, 12, 6, 10, 0, 'CET') - to_datetime(2000, 5, 12, 6, 10, 0, 'KST');
```

中欧时间 2020 年 5 月 12 日上午 6:10 减去韩国标准时间 2000 年 5 月 12 日上午 6:10。结果是：`{<duration>'175328:00:00'}`。

现在，我们再次尝试让乔纳森从德古拉城堡中逃脱。时间是 5 月 12 日上午 10:35。同一天，伦敦的早上 6:10，米娜正在喝着早茶。这两个事件之间相差多少秒？他们处于不同的时区，但我们不需要亲自计算；我们只需指定时区，剩下的工作将由 EdgeDB 完成：

```edgeql
select to_datetime(2020, 5, 12, 10, 35, 0, 'EEST') - to_datetime(2020, 5, 12, 6, 10, 0, 'UTC');
```

答案是 1 小时 25 分钟：`{<duration>'1:25:00'}。

为了使查询语句更加可读，我们还可以使用关键字 `with` 来创建变量。然后我们可以在下面的 `select` 中使用这个变量。我们将创建两个变量，分别叫做 `jonathan_wants_to_escape` 和 `mina_has_tea`，然后从其中一个中减去另一个以获得 `duration`。有了变量名，我们现在要做的事情就显得更加清楚了：

```edgeql
with
  jonathan_wants_to_escape := to_datetime(2020, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := to_datetime(2020, 5, 12, 6, 10, 0, 'UTC'),
select jonathan_wants_to_escape - mina_has_tea;
```

输出结果是一样的：`{<duration>'1:25:00'}`。只要我们知道时区，当我们需要一个 `duration` 时，`datetime` 类型就可以胜任。

## 时间跨度类型 duration

除了对两个 `datetime` 进行相减，你也可以直接转换出一个 `duration`。为此，只需写下数字及对应的单位：`microseconds`，`milliseconds`，`seconds`，`minutes`，或 `hours`（“微秒”、“毫秒”、“秒”、“分钟”或“小时”）。EdgeDB 将返回一个秒数或更精确的单位。比如 `select <duration>'2 hours';` 将返回 `{<duration>'2:00:00'}`；`select <duration>'2 microseconds';` 将返回 `{<duration>'0:00:00.000002'}`。

你也可以包含多个单位。例如：

```edgeql
select <duration>'6 hours 6 minutes 10 milliseconds 678999 microseconds';
```

这将返回：`{<duration>'6:06:00.688999'}`。

在做 `duration` 的转换时，EdgeDB 在输入方面是非常包容的，并且会忽略复数和其他符号。因此，即使是这种可怕的输入也可以工作（但我们并不推荐这样做）：

```edgeql
select <duration>'1 hours, 8 minute ** 5 second ()()()( //// 6 milliseconds' -
  <duration>'10 microsecond 7 minutes %%%%%%% 10 seconds 5 hour';
```

结果是：`{<duration>'-3:59:04.99401'}`。

## 必需的链接

现在我们需要为三个女性吸血鬼创建一个类型。我们称它为 `MinorVampire`。它将有一个指向 `Vampire` 类型的 `required` 链接。这是因为她们被德古拉控制着，她们只因为德古拉的存在而作为 `MinorVampire`（小鬼）存在。

```sdl
type MinorVampire extending Person {
  required master: Vampire;
}
```

现在因为 `master` 是必需的，我们不能插入一个只有名字的 `MinorVampire`。如果那样做，我们会得到错误：`ERROR: MissingRequiredError: missing value for required link default::MinorVampire.master`。因此，让我们插入小鬼们的数据时，并将其 `master` 链接到德古拉（Dracula）：

```edgeql
insert MinorVampire {
  name := 'Vampire Woman 1',
    master := assert_single(
    (select Vampire filter .name = 'Count Dracula')
  ),
};
```

如前面所学到的，你需要将获取“Count Dracula”数据的查询语句放到圆括号中，然后将其放入 `assert_single()` 函数中。该函数会检查确保查询结果中只有一个元素。这是必要的，因为 EdgeDB 不知道我们是否只有一个叫做德古拉伯爵（Count Dracula）的 `Vampire`，但这里我们只需要提供一个吸血鬼作为“master”（请记住：`required` link 是 `required single` link 的缩写）。如果我们在没有 `assert_single()` 函数的情况下尝试上面的语句，我们将会得到以下错误：

```
error: possibly more than one element returned by an expression for a link 'master' declared as 'single'
```

要注意：用 `assert_single()` 查询时，如果返回了两个或两个以上元素的话，则会出现 `CardinalityViolationError`。因此最好在你确认只会返回一个元素时才使用 `assert_single()` 函数。

## 关键词：describe

`MinorVampire` 类型扩展自 `Person`，`Vampire` 也是如此。类型可以继续扩展其他类型，并且可以同时扩展多个类型。你这样做的次数越多，试图在脑海中将它们组合在一起就越困难。这正是 `describe` 可以提供帮助的地方，它可以准确地显示出任何类型的组成内容。具体有以下三种方法：

- `describe type MinorVampire`：这将给出类型的 DDL 描述 {ref}`DDL (data definition language) <docs:ref_eql_ddl>`。DDL 是比 SDL（我们一直在使用的语言）更低级别的语言。对于架构（schema），它不太方便，但更明确，可用于快速更改。我们不会在本课程中系统学习 DDL，但稍后你可能会发现有时它很有用。例如，使用它你可以快速创建函数而无需进行 _显式的（explicit）_ 迁移（migration，这里的 migration 指的是一次完整的、常规的 schema 变更）。如果你了解 SDL，则不难掌握 DDL 中的一些技巧。

(请注意上面提到的 _显式的（explicit）_ 一词：使用 DDL 仍会导致迁移，只是 _隐式的（implicit）_ 迁移。换句话说，迁移发生时并未将其称为迁移。这是一种快又脏的更改方式，但在大多数情况下，使用 SDL 架构的适当迁移工具仍是首选方式。）

现在让我们回到 `describe type`，在 DDL 中给出的结果。下面是 `MinorVampire` 类型的样子：

```
{
  'CREATE TYPE default::MinorVampire EXTENDING default::Person {
    CREATE REQUIRED LINK master: default::Vampire;
};',
}
```

`create` 关键字表明它是一系列的快速命令，这也是顺序很重要的原因。换句话说，SDL 是 _陈述的（declarative）_（它 _陈述_ 将是什么，而不用担心顺序），而 DDL 是 _命令的（imperative）_（它是一系列更改状态的命令）。此外，它只显示了创建它的 DDL 命令，没有向我们显示它扩展的所有 `Person` 的链接和属性，这不是我们想要的。下一个方法是：

- `describe type MinorVampire as sdl`：同样的事情，但用 SDL 表达。

输出也几乎相同，只是上一个方法输出结果的 SDL 版本。对于我们现在想要的信息，这也不够：

```
{
  'type default::MinorVampire extending default::Person {
    required link master: default::Vampire;
};',
}
```

你会注意到它与我们的 SDL 架构（SDL schema）基本相同，只是更加冗长和详细：`type default::MinorVampire` 替代了 `type MinorVampire` 等等。

- 第三个方法是 `describe type MinorVampire as text`。这就是我们想要的，因为它显示了类型内部的所有内容，包括它所扩展的类型中的内容。这是输出：

```
{
  'type default::MinorVampire extending default::Person {
    required single link __type__: schema::Type {
        readonly := true;
    };
    optional single link lover: default::Person;
    required single link master: default::Vampire;
    optional multi link places_visited: default::Place;
    required single property id: std::uuid {
        readonly := true;
    };
    required single property name: std::str;
};',
}
```

（请注意：`as text` 不包括约束和注释。要查看这些，请在末尾添加 `verbose`：`describe type MinorVampire as text verbose;`。你将在第 14 章中学习注释相关内容。）

`readonly := true` 的部分我们不必在意，它们是自动生成的（我们无法对它们做什么）。对于其他部分，可以看出我们必须有一个 `name` 和一个 `master`，且可以选择为这些 `MinorVampire` 添加 `lover` 和 `places_visited`。

此外，你还可以通过键入 `describe schema` 或 `describe module default`（如果需要，可以使用 `as sdl` 或 `as text` ）获得 _很长_ 的输出，即显示我们迄今为止构建的整个架构。

因此，对于类型，我们把 `TYPE` 放到 `describe` 后面；对于模块，我们把 `module` 放到 `describe` 后面，那么对于链接或其他的呢？以下是可以放到 `describe` 后面的所有关键字的列表：`object`、`annotation`、`constraint`、`function`、`link`、`module`、`property`、`scalar type`、`type`。如果你不想全部记住它们，只需使用 `object`：它会匹配你的架构（schema）中的任何内容（模块除外）。

[→ 点击这里查看到第 5 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 你认为 `select to_datetime(3600);` 将返回什么？为什么？

   提示：检查上面的函数签名，看看当你输入 3600 时，EdgeDB 会选择哪一个。

2. `select <int16>9 + 1.06n is decimal;` 能工作吗？如果可以，它将返回 `{true}` 吗？

3. 从 2003 年土库曼斯坦 (TMT) 圣诞节的早上 5:00 到乌兹别克斯坦 (UZT) 同年新年除夕夜的晚上 7:00 之间过去了多少秒？

4. 如何用对上题中的两个时间使用 `with` 写出同样查询效果的查询语句？

5. 如果你只想查看如何编写的某个类型，那么描述该类型的最佳方式是什么？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _一位女吸血鬼对她的姐妹们说：“他年轻且强壮；我们所有人都有亲吻……”_
