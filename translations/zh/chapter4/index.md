---
tags: Local Time, Advanced Filtering
leadImage: illustration_04.jpg
---

# 第四章 - "德古拉伯爵真是个奇怪的人"

> 乔纳森·哈克（Jonathan Harker）起晚了，整个白天只有他一个人独自待在城堡里。夜幕降临后，德古拉（Dracula）出现了，他们聊了 **一个通宵（through the night）**。德古拉正在制定搬到伦敦的计划，乔纳森（Jonathan）给了他一些关于购房的建议。他告诉德古拉，有个名为 Carfax 的别墅是个值得购买的好房子。房子很大且安静。附近有一家疯人院，但也不是很近。德古拉认为这是个不错的主意。此外，德古拉告诉乔纳森不要进入城堡中任何上了锁的房间，因为里面很危险。这时乔纳森发现已经快到早上了——他们又聊了整整一夜。德古拉突然站起来说他必须走了，随后便离开了房间。乔纳森又想起了已经回到了伦敦的 **米娜（Mina）** 还在等他回去完婚。同时，他也开始觉得德古拉和他的城堡有些问题。说真的，其他人在哪呢？

现在让我们来为乔纳森的女朋友——米娜·默里（Mina Murray）创建一条数据。但在此之前，我们需要先在 `Person` 类型的结构中增加一个指向 `Person` 类型的链接，并命名为 `lover`：

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Places;
  lover: Person;
}
```

这样就可以把乔纳森和米娜关联在一起了。我们假设一个人只能有一个 `lover`，所以它是一个 `single link`。

米娜目前在伦敦，且我们还不知道她是否去过其他什么地方。所以我们先快速地为“伦敦”插入一条数据。这很简单：

```edgeql
insert City {
    name := 'London',
};
```

为了把伦敦写入米娜造访过的城市，我们只需要做一个快速的查询：`select City filter .name = 'London'`。这将为米娜提供与 `.name = 'London'` 匹配的 `City`；如果该城市不存在，这里也不会给出错误：它只会返回一个空集 `{}`。

## 关键词：分离、存在及限制

现在我们来创建一个名为“Mina Murray”的 NPC，其中 `lover` 的赋值部分稍微复杂一些：

```edgeql
insert NPC {
  name := 'Mina Murray',
  lover := assert_single(
    (
      select detached NPC
      filter .name = 'Jonathan Harker'
    )
  ),
  places_visited := (select City filter .name = 'London'),
};
```

这里，你可能已经注意到了两个新的内容：

- `detached`：使用它是因为我们在一个 `NPC` 的 `insert` 语句里想要链接到另一个相同类型的 `NPC`。因此，我们需要添加 `detached` 来指明此处我们正在 `select` 的是非特定的普通 `NPC`，不是这里正在创建的 `NPC`。
- {eql:func}`docs:std::assert_single`：使用它是因为 `lover` 是个 `single link`。 EdgeDB 并不知道执行 `select detached NPC filter .name = 'Jonathan Harker'` 会得到多少结果，可能有 2 个或 3 个或更多的 `Jonathan Harkers`。为了确保只会得到一个结果来创建 `single link`，我们使用函数 `assert_single()`。

现在，我们想查询一下谁是单身，谁不是。我们可以通过创建一个新的变量并用 `:=` 来赋予它一个“计算的（computed）”属性来实现。首先，我们先来看一下下面这个常规的查询：

```edgeql
select Person {
  name,
  lover: {
    name
  }
};
```

输出结果为：

```
{
  default::NPC {name: 'Jonathan Harker', lover: {}},
  default::NPC {name: 'The innkeeper', lover: {}},
  default::NPC {name: 'Mina Murray', lover: default::NPC {name: 'Jonathan Harker'}},
  default::PC {name: 'Emil Sinclair', lover: {}},
  default::Vampire {name: 'Count Dracula', lover: {}},
}
```

不难看出，米娜·默里（Mina Murray）有一个爱人，是乔纳森，但乔纳森·哈克（Jonathan Harker）却还没有爱人，这是因为我们之前创建乔纳森的时候还没有 `lover` 这个属性。我们将会在后续的第 6、14、15章中学习一些技巧来处理这种情况。这里我们就先暂时保持乔纳森·哈克的 `lover` 链接是 `{}`。

回到查询语句：如果我们只是想根据角色是否有爱人来返回 `true` 或 `false`，该怎么办呢？我们可以在查询语句中添加一个使用 `exists` 的“计算（computed）”属性。如下所示，如果 `Person.lover` 返回的是一个有内容的集合，则 `exists` 将返回 `true`；如果它返回的是 `{}`（即什么都没有），则 `exists` 返回 `false`。这再次说明 EdgeDB 中没有 null。由此，我们将查询语句调整为：

```edgeql
select Person {
  name,
  is_single := not exists Person.lover,
};
```

这次输出的结果是：

```
{
  default::NPC {name: 'Jonathan Harker', is_single: true},
  default::NPC {name: 'The innkeeper', is_single: true},
  default::NPC {name: 'Mina Murray', is_single: false},
  default::PC {name: 'Emil Sinclair', is_single: true},
  default::Vampire {name: 'Count Dracula', is_single: true},
}
```

这里也展示了抽象类型的一个重要作用：我们通过对 `Person` 的快速搜索即可得到来自 `Vampire`、`PC` 和 `NPC` 的所有数据，因为它们都来自 `abstract type Person`。

我们得到了很多“单身狗”，现在让我们尝试只选择其中一个：

```edgeql
select Person {
  name,
  is_single := not exists Person.lover,
} filter .is_single limit 1;
```

输出结果为：
```
{default::Vampire {name: 'Count Dracula', is_single: true}}
```

请注意，这里我们使用了 `limit 1` 而不是 `assert_single()`，因为我们只是想要其中随便一个结果，我们并不在乎是否还有更多满足我们 `filter` 的结果。此处，如果使用 `assert_single()`，会因为数据库中存在多个结果而报错。当然，如果我们想获得有最大数量限制的一系列对象，可以使用 limit 2、limit 3 等等。

可能上面的查询中实际上并没有选择“Count Dracula”，而是选择了其他某个 `Person`。`limit 1` 最多选择出一个结果，但它并没有指明会是哪个，只是数据库中找到的第一个结果。我们将会在 [第 10 章](../chapter10/index.md) 中介绍选择查看结果的顺序。

我们还可以将计算属性放在类型本身中。下面使用了与上面所提相同的计算属性，这里只是将它运用在了 `Person` 类型的定义中：

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Places;
  lover: Person;
  property is_single := not exists .lover;
}
```

现在，你可能对“计算的（computed）”链接及属性在后端数据库中的表示方式感到好奇。它们很有趣，因为它们不会出现在实际数据库中 {ref}`don't show up in the actual database <docs:ref_datamodel_computed>`，他们仅会在你查询时出现。你不必指定它的类型，因为“计算的（computed）”表达式自身就决定了类型。比如，当你查看带有快速计算变量的查询时，如 `select country_name := 'Romania'`，每次执行查询时 EdgeDB 都会计算 `country_name`，且类型会被确定为字符串。类型定义中的“计算（computed）”链接及属性所做的事情其实是一样的。但无论如何，他们用起来同常规的属性及链接并无二致，因为计算属性及链接的公式在定义类型的时候就写死了。换言之，计算属性与常规属性的底层实现虽有不同，但对使用者来说都是一样的属性。

## 报时的方法

现在我们来学习时间，这对于我们的游戏也是十分重要的。请记住，吸血鬼只能在夜幕降临后出去（因为他们害怕阳光）。

乔纳森·哈克（Jonathan Harker）所在的罗马尼亚的平均日出时间是 7 am，日落时间是 7 pm。随着季节不同会有所变化，但为了简单起见，我们将仅使用早上 7 点和晚上 7 点来决定是白天还是黑夜。

EdgeDB 有两种主要的时间类型。

- `std::datetime`：这是非常精确的并且总带有一个时区。`datetime` 中的时间使用 ISO 8601 标准。
- `cal::local_datetime`：这个不会考虑时区。

此外，还有两个与 `cal::local_datetime` 类似的时间类型：

- `cal::local_time`：用于当你只需要知道一天内的时间时；
- `cal::local_date`：用于当你只需要知道年、月、日时。

我们先从 `cal::local_time` 开始。

`cal::local_time` 是很容易创建的，因为你可以从“HH:MM:SS”格式的 `str` 转换为它：

```edgeql
select <cal::local_time>('15:44:56');
```

执行后输出：

```
{<cal::local_time>'15:44:56'}
```

假设我们的游戏中有一个时钟，它以 `str` 的形式给出时间，就像上面例子中的 '15:44:56'。现在让我们来创建一个 `Time` 类型，它对后续会很有帮助：

```sdl
type Time {
  required clock: str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
}
```

`.clock[0:2]` 是使用“切片” {eql:op}`"slicing" <docs:arrayslice>` 的一个例子。[0:2] 表示从索引 0（第一个索引）开始，并到索引 2 _之前_ 停止，即取出索引 0 和 1。这也说明当你要将 `str` 转换为 `cal::local_time` 时，你必须要用两个数字字符来表示小时（即：09 可以，但 9 不行）。

所以下面这个语句是无法正常工作的：

```edgeql
select <cal::local_time>'9:55:05';
```

将会产生错误：

```
ERROR: InvalidValueError: invalid input syntax for type cal::local_time: '9:55:05'
  Hint: Please use ISO8601 format. Examples: 18:43:27 or 18:43 Alternatively "to_local_time" function provides custom formatting options.
```

由此我们可以确信索引 0 到 2 的切片可以为我们提供两个数字来表示一天中某个时间的小时数。

现在使用这个 `Time` 类型，我们可以通过执行以下操作来获取小时数。首先插入时间：

```edgeql
insert Time {
    clock := '09:55:05',
};
```

然后我们可以 `select` 我们的 `Time` 对象和里面的所有东西：

```edgeql
select Time {
  clock,
  clock_time,
  hour,
};
```

执行后，我们得到了期望的输出，它展示了所有，包括小时：

`{default::Time {clock: '09:55:05', clock_time: <cal::local_time>'09:55:05', hour: '09'}}`

最后，我们可以在 `Time` 类型中添加一些逻辑来查看吸血鬼是醒着还是睡了。这里我们可以使用一个 `enum`，但简单起见，我们将它设为一个 `str`。

```sdl
type Time {
  required clock: str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
        else SleepState.Awake;
}
```

因此，`vampires_are` 是如下这样计算的：

- 首先 EdgeDB 会查看取到的小时的数值是否大于 7 且小于 19（晚上 7 点）。这里用两个数字进行比较比用数字与字符串进行比较要好，因此我们编写 `<int16>.hour` 而不是 `.hour`，这样就可以通过类型转换达到用数字与数字进行比较的目的。
- 然后 EdgeDB 会基于比较结果给出一个字符串来说明现在的状态是“睡着（'asleep'）”还是“醒着（'awake'）”。

现在，如果我们对所有属性进行 `select`，我们将得到：

`{default::Time {clock: '09:55:05', clock_time: <cal::local_time>'09:55:05', hour: '09', vampires_are: Asleep}}`

关于 `else`，这里有一个注意事项：你可以在 `(result) if (condition) else` 格式中根据需要多次使用 `else`。例如：

```
property vampires_are := SleepState.JustWakingUp if <int16>.hour = 19 else
                  SleepState.GoingToBed if <int16>.hour = 6 else
                  SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19 else
                  SleepState.Awake;
```

## 插入的同时做选择

回到第 3 章，我们学习了如何在删除的同时做选择。对于 `insert`，你可以做同样的事情。与任何其他 `select` 一样，你可以通过将 `insert` 括在圆括号中然后选择它来执行相同的操作。因为当我们插入一个新的 `Time` 时，我们只会得到一个 `uuid`：

```edgeql
insert Time {
  clock := '22:44:10'
};
```

其输出为：`{default::Time {id: 3f6951c6-ff48-11eb-915e-3fd1092b2757}}`

现在，将整个输入包装在 `select ()` 中，这样我们就可以在插入它时显示它其中的属性了。因为是括号括起来的，所以 EdgeDB 会先做括号里的操作，然后再用选择做一个普通的查询。除了要显示的属性外，我们还可以添加“计算的（computed）”属性。下面，让我们来试一下：

```edgeql
select ( # Start a selection
  insert Time { # Put the insert inside it
    clock := '22:44:10'
  }
) # The bracket finishes the selection
  { # Now just choose the properties we want
    clock,
    hour,
    vampires_are,
    double_hour := <int16>.hour * 2
  };
```

现在的输出结果对我们来说更有意义了，即：`{default::Time {clock: '22:44:10', hour: '22', vampires_are: Awake, double_hour: 44}}`。我们知道了时间（clock）和小时（hour），同时也了解到了吸血鬼是醒着的，我们甚至可以对我们刚刚输入的对象做些其他计算，如示例中的 `double_hour`。

[→ 点击这里查看到第 4 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 下面的插入语句无法正常工作：

   ```edgeql
   insert NPC {
     name := 'I Love Mina',
     lover := assert_single(
       (select NPC filter .name like '%Mina%')
     )
   };
   ```

   报错是：`invalid reference to default::NPC: self-referencing inserts are not allowed`。我们可以使用什么关键字来使这个插入语句工作呢？

   另外，我们也可以使用另一种方法来使其在没有添加关键字的情况下工作。你能想到别的办法吗？

2. 如何显示名称里包含字母 `a` 的 `Person` 对象，且最多显示 2 个（以及它们的 `name` 属性）？

3. 如何显示出从未访问过任何地方的 `Person` 对象（以及它们的 `name` 属性）？

   提示：获取 `.places_visited` 返回 `{}` 的所有 `Person` 对象。

4. 假设你有以下 `cal::local_time` 类型：

   ```edgeql
   select has_nine_in_it := <cal::local_time>'09:09:09';
   ```

   这会显示：`{<cal::local_time>'09:09:09'}`。如何改为：时间里有 9 则显示 {true}，没有则显示 {false}？

5. 我们现在插入一个名为“The Innkeeper's Son”的角色：

   ```edgeql
   insert NPC {
     name := "The Innkeeper's Son",
     age := 10
   };
   ```

   你将如何在插入的同时，用 `select` 来显示它的 `name`，`age` 和 `age_ten_years_later`？`age_ten_years_later` 是指 `age` 加 10。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森愈发好奇：城堡里有什么？_
