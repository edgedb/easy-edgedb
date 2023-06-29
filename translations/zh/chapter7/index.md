---
tags: Constraint Delegation, $ Parameters
---

# 第七章 - 乔纳森最终“离开”了城堡

> 乔纳森（Jonathan）白天偷偷溜进了德古拉（Dracula）的房间，看见他睡在棺材里。现在他终于知道德古拉是一只吸血鬼了。过了几天，德古拉说他明天就要走了。乔纳森认为这是一个好机会，并要求他让自己可以马上离开。德古拉打开门并回应道：“好吧，如你愿意……” 但是外面有很多狼，它们嚎叫着，发出很大的声音。德古拉又道：“你可以离开了，再见！” 乔森纳知道，如果他走出去，就会被狼群干掉。他也知道这是德古拉召唤的狼群，于是他请德古拉把门关上。德古拉微笑着关上了门……乔纳森被困住了。之后，乔纳森听到德古拉告诉那三个女吸血鬼可以在他明天离开后享用乔纳森。第二天，德古拉的朋友把德古拉带走了（用棺材），乔纳森独自一人留在城堡里……不久，夜幕降临。因为所有的门都上了锁，他决定从窗户爬出去，因为他宁愿摔死，也不愿意与女吸血鬼们共处一室。他在日记中写道“再见了，各位！再见了，米娜！” ，然后他开始往外爬。

## 更多约束

在乔纳森爬墙时，我们可以继续处理我们的数据库架构（schema）。在我们的书中，并没有相同名字的角色，所以应该只有一个米娜·默里（Mina Murray），一个德古拉伯爵等等。这是在 `Person` 类型的 `name` 上放置“约束” {ref}`constraint <docs:ref_datamodel_constraints>` 的好时机，以确保我们不会有重复姓名的对象被插入。`constraint` 是一个限制，我们在人类的 `age` 中已经看到了最多只能活到 120 岁的限制。对于 `name`，我们可以给它增加一个名为 `constraint exclusive` 的限制，以防止两个相同类型的对象具有相同的名称。具体操作如下所示，即在属性后面的块中放置一个 `constraint`：

```sdl
abstract type Person {
  required name: str { ## Add a block
      constraint exclusive;       ## and the constraint
  }
  multi places_visited: Place;
  lover: Person;
}
```

现在我们确保了将只有一个 `Jonathan Harker`，一个 `Mina Murray` 等等在我们的数据库里。而在现实生活中，这对那些类似邮箱地址、用户 ID 等我们希望具有唯一性的属性十分有用。在我们的数据库里，我们也将给 `Place` 里的 `name` 添加 `constraint exclusive`，因为这些地方的名字也是唯一的：

```sdl
abstract type Place {
  required name: str {
      constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

## 传递约束

现在，我们的 `Person` 类型的 `name` 属性具有 `constraint exclusive`，因此任何扩展自 `Person` 的类型都不可以拥有同样的名称。这对我们的游戏来说并无大碍，因为我们已经知道了书中所有角色的名称，且不打算制作任何真正的 `PC` 类型（游戏玩家）的对象。但是如果我们稍后想创建一个名为“Jonathan Harker”的 `PC` 该怎么办？现在是不允许的，因为我们已经有了一个同名的 `NPC`，且 `NPC` 里的 `name` 同样来自 `Person` 类型。

幸运的是这里有一个简单的办法绕过它：就是在 `constraint` 前面添加关键词 `delegated`。这将“委托（delegates）”（传递）约束给子类型，因此排他性检查将分别针对 `PC`、`NPC`、`Vampire` 等进行（而不在他们彼此之间进行检查），你只是需要额外加上关键字 `delegated` 在之前的列子当中：

```sdl
abstract type Person {
  required name: str {
    delegated constraint exclusive;
  }
  multi places_visited: Place;
  lover: Person;
  strength: int16;
}
```

有了它，对于扩展自 `Person` 的类型，如 `PC`、`NPC`、`Vampire` 等，分别可以拥有最多一个叫做“Jonathan Harker”的对象。

同样，我们也可以将 `delegated constraint` 应用到 `Place`，比如我们允许某个 `Country` 可以和某个 `City` 同名，那么我们就可以将 `Place` 类型里的 `name` 进行如下的更新：

```sdl
abstract type Place {
  required name: str {
      delegated constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

## 在查询中使用函数

现在让我们来考虑一下我们的游戏机制。书里说城堡里的门对于乔纳森来说太难打开了，但是德古拉足够强壮，可以打开所有。在真正的游戏中，它会更复杂，但我们可以尝试用一些简单的方法来模仿这个事实：

- 门有力量（strength），人也有力量。
- 如果人的力量大于门，则人可以打开门。

因此我们将创建一个 `Castle` 类型，并给它装上一些门（即设置属性 `doors`）。现在我们想分别给这些门设置一个表示“强度（strength）”的数字，所以我们将 `doors` 的类型设为 `array<int16>`：

```sdl
type Castle extending Place {
    doors: array<int16>;
}
```

然后我们假设这里有三个主要的门用来出入德古拉城堡，所以我们按如下方式 `insert` 它们：

```edgeql
insert Castle {
  name := 'Castle Dracula',
  doors := [6, 19, 10],
};
```

然后我们再添加一个 `strength: int16;` 到 `Person` 类型。这个属性将不是必需的，因为我们并不知道本书中所有人的力量……当然，如果游戏需要我们可以在之后将其设置为 `required`。

现在我们给乔纳森（Jonathan）一个等于 5 的力量值。像之前一样，使用 `update` 和 `set` 进行更新：

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  strength := 5
};
```

我们知道乔纳森无法冲出城堡，那么让我们尝试使用一个查询语句来展示这个事实。要做到冲出城堡，他需要拥有比门还大的力量。或者换句话说，他需要比最不牢固的门拥有更大的力量。

幸运的是，有一个叫做 `min()` 的函数可以给出一个集合中的最小值，因此我们可以利用它。如果乔纳森的力量大于拥有最小力量的门的数值，他则可以逃脱。下面的查询看起来应该可以工作，但实则不然：

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  castle_doors := (select Castle filter .name = 'Castle Dracula').doors,
select jonathan_strength > min(castle_doors);
```

这里会报错：

```
error: operator '>' cannot be applied to operands of type 'std::int16' and 'array<std::int16>'
```

我们可以通过查看 {eql:func}`min() <docs:std::min>` 函数的签名来发现问题：

```
std::min(values: set of anytype) -> optional anytype
```

重要的部分是 `set of`：它需要的是一个集合，所以我们需要用大括号括起来。但我们不能只在数组前后放置大括号，因为这样它就会变成一个数组的集合（比如 {[5, 6], [7]}）。所以 `select min({[5, 6]});` 会返回 `{[5, 6]}`，而不是 `{5}`，因为 `{[5, 6]}` 里只有一个数组，所以 `{}` 里最小的数组只能是 `[5, 6]`。

这也意味着 `select min({[5, 6], [2, 4]});` 将会返回 `{[2, 4]}`（而不是 2）。这不是我们想要的，所以不能简单地改为 `select jonathan_strength > min({castle_doors});`。

实际上，我们想要使用的是 {eql:func}` ``array_unpack()`` <docs:std::array_unpack>` 函数，它接受一个数组并可以将其解包为一个集合。所以我们将对 `castle_doors` 使用该函数：

```edgeql
with
  jonathan := (select Person filter .name = 'Jonathan Harker'),
  castle := (select Castle filter .name = 'Castle Dracula'),
  select jonathan.strength > min(array_unpack(castle.doors));
```

于是，我们将得到 `{false}`。很好！现在我们成功展示了乔纳森不能打开任何门的事实。他将不得不从窗户爬出去。

除了 `min()`，当然还有 `max()`。`len()` 和 `count()` 等也都很有用：`len()` 可以给出一个对象的长度，`count()` 可以给出它们的数量。下面是一个使用 `len()` 获取所有 `NPC` 对象名称长度的示例：

```edgeql
select (NPC.name, 'Name length is: ' ++ <str>len(NPC.name));
```

别忘了我们需要做一个 `<str>` 的类型转换，因为 `len()` 返回的是一个整数，且 EdgeDB 无法级联 `++` 一个字符串和一个整数。结果如下：

```
{
  ('The innkeeper', 'Name length is: 13'),
  ('Mina Murray', 'Name length is: 11'),
  ('Jonathan Harker', 'Name length is: 15'),
}
```

另一个使用 `count()` 的例子也需要做 `<str>` 的类型转换：

```edgeql
select 'There are ' ++ <str>(select count(Place) - count(Castle)) ++ ' more places than castles';
```

结果是：`{'There are 8 more places than castles'}`

在之后的几章中，我们将学习如何创建自定义的函数来缩短查询时间。

## 使用 $ 设置参数

假设我们经常需要使用下面的查询语句来查找满足条件的 `City` 对象：

```edgeql
select City {
  name,
  modern_name
} filter .name ilike '%i%' and exists (.modern_name);
```

上面语句可以正常工作并返回一个城市：`{default::City {name: 'Bistritz', modern_name: 'Bistrița'}}`。

但每次查找，都可能需要更改最后一行的过滤内容，这可能有点烦人，即在我们再次按 Enter 执行新的查询语句之前，会因为删除和重新输入过滤条件需要很多的光标移动。

这正是介绍如何用 `$` 向查询添加参数的好时机。我们可以给参数起一个名字，EdgeDB 会在每次查询中询问我们要给它什么数值。现在让我们从一些非常简单的事情开始：

```edgeql
select City {
  name
} filter .name = 'London';
```

现在让我们把 `'London'` 改为 `$name`。注意：这照样不会工作，猜猜为什么？

```edgeql
select City {
  name
} filter .name = $name;
```

问题在于 `$name` 可以是任何，EdgeDB 不知道它将是什么类型。错误提示为：`error: missing a type cast before the parameter`。所以，因为它是一个字符串，我们将使用 `<str>` 进行转换：

```edgeql
select City {
  name
} filter .name = <str>$name;
```

执行后，我们会收到一个提示 `Parameter <str>$name:`，要求我们输入数值。输入 `London`，不需要引号，因为 EdgeDB 已经知道它是一个字符串了。于是得到结果：`{default::City {name: 'London'}}`。

现在让我们使用两个参数来创建一个更复杂（和有用）的查询。我们将它们称为 `$name` 和 `$has_modern_name`。不要忘记对它们进行类型转换：

```edgeql
select City {
  name,
  modern_name
} filter
    .name ilike '%' ++ <str>$name ++ '%'
  and
    exists (.modern_name) = <bool>$has_modern_name;
```

由于有两个参数，EdgeDB 会要求我们输入两个值。下面是一个示例：

```
Parameter <str>$name: b
Parameter <bool>$has_modern_name: true
```

因此，这将会输出名称中包含“b”，且拥有一个（不同于旧称的）现代别名的所有 `City` 类型的对象。执行结果是：

```
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

参数在插入语句中也同样有效。下面是一个 `Time` 插入，提示用户输入小时、分钟和秒：

```edgeql
select (
  insert Time {
    clock := <str>$hour ++ <str>$minute ++ <str>$second
  }
) {
  clock,
  clock_time,
  hour,
  vampires_are
};
Parameter <str>$hour: 10
Parameter <str>$minute: 09
Parameter <str>$second: 09
```

输出结果是：

```
{
  default::Time {
    clock: '100909',
    clock_time: <cal::local_time>'10:09:09',
    hour: '10',
    vampires_are: Asleep,
  },
}
```

请注意，类型转换意味着你可以只是输入 `10`，而不用输入 `'10'`。

那么，如果你希望参数是 _可选的_ 呢？没问题，只需将 `optional` 放在类型名称之前（置入 `<>` 括号内）。因此，如果你希望所有内容都是可选的，则上面的插入语句将变更为：

```edgeql
select (
  insert Time {
    clock := <optional str>$hour ++ <optional str>$minute ++ <optional str>$second
  }
) {
  clock,
  clock_time,
  hour,
  awake
};
```

当然，`Time` 类型的 `clock` 属性需要正确的格式，所以上面的做法不是一个好主意。这里只是为了展示一下你可以如何做。

`optional` 的相反是 `REQUIRED`，但因为它是默认的，所以你不需要总是写明它。

我们在之前的章节里学到的关键字 `update` 也可以使用参数，所以总共有四个关键字我们可以使用参数，它们是：`select`、`insert`、`update` 和 `delete`。

[→ 点击这里查看到第 7 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何选择出每一个 City 及他们名字的长度？

2. 如何选择出每一个 City 并展示其 `name` 长度减去 `modern_name` 长度的结果，如果 `modern_name` 不存在，则显示 0。 

3. 如果在上一题中想用 `'Modern name does not exist'` 替代 0，作为 `modern_name` 不存在时的结果显示，该如何做？

4. 如果已经有 7 个 NPC 存在，你将如何插入名为“NPC number 8”的 NPC？

5. 如何选择出名字最短的 `Person` 类型的对象？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _瓦尔纳市的工人将箱子装上了船，德古拉就在其中……_
