---
tags: Constraints, Deleting
leadImage: illustration_03.jpg
---

# 第三章 - 乔纳森前往德古拉城堡

这一章，我们将会开始考虑有关“时间”的一些概念。那么，乔纳森·哈克（Jonathan Harker）在本章到底又做了些什么呢？

> 乔纳森·哈克（Jonathan Harker）乘坐马车翻山越岭终于抵达了德古拉城堡（Castle Dracula）。他的旅途十分糟糕，因为到处都是雪、奇怪的蓝色火焰和狼群。乔纳森抵达时，已经是晚上了。见到了德古拉伯爵（Count Dracula）后，他们聊了整整一夜。但在太阳升起之前，德古拉就离开了，因为吸血鬼会被阳光灼伤。日子一天天过去了，乔纳森仍然不知道德古拉是只吸血鬼。但他确实注意到了一些奇怪的事情，比如整座城堡似乎都空无一人。如果德古拉那么富有，那么他的仆人都在哪里呢？桌子上的饭菜是谁做的呢？但乔纳森被德古拉的那些有趣的历史故事所吸引，因此到目前为止他仍然很享受这次旅行。

现在我们已经完全进入了德古拉的城堡，所以是时候创建一个 `Vampire`（吸血鬼）类型了。我们可以基于 `abstract type Person` 扩展出它，因为该类型有 `name` 和 `places_visited`，它们也同样适用于 `Vampire`。但是吸血鬼与人类不同，它们可以永生（不会衰亡）。因此，这里我们可以添加 `age` 到 `Person`，以便所有由其扩展出的类型也可以使用它。那么 `Person` 的定义将变为：

```sdl
abstract type Person {
  required name: str;
  multi places_visited: City;
  age: int16;
}
```

`int16` 表示 16 位（2 字节）整数，从 -32768 到 +32767。对于年龄来说，这绰绰有余，所以我们不需要用更大的 `int32` 或 `int64` 类型。我们也不希望它成为“必需属性” `required` property，因为我们并不关心每一个人的年龄。

但是我们不会期待 `PC` 和 `NPC` 活到 32767 岁，所以我们现在只给 `Vampire` 提供 `age`，之后我们再考虑在其他类型上用 `age`（因此，我们从上面的 `Person` 定义中先挪去 `age`)。于是，我们从 `Person` 扩展出 `Vampire` 类型，并对其添加属性 `age`：

```sdl
type Vampire extending Person {
  age: int16;
}
```

现在，我们可以创建德古拉伯爵（Count Dracula）了。我们知道他住在罗马尼亚（Romania），但这不是一个城市。因此是时候调整一下 `City`（城市）类型了。我们将其更名为 `Place`（地点），并将其改为 `abstract type`（抽象类型），然后基于其扩展出 `City` 类型。我们还将添加一个有类似作用的 `Country` 类型。现在这些类型的定义如下所示：

```sdl
abstract type Place {
  required name: str;
  modern_name: str;
  important_places: array<str>;
}

type City extending Place;

type Country extending Place;
```

我们需要将 `Person` 类型里的 `places_visited` 由 `City` 修改为 `Place`。毕竟，角色们不仅可以访问城市，还可以访问更多其他类型的地方：

```sdl
abstract type Person {
  required name: str;
  multi places_visited: Place;
}
```

现在，创建一个 `Country` 很容易，只需插入并给它一个名字。接下来，快速插入匈牙利（Hungary）和罗马尼亚（Romania）两个 `Country` 类型的对象：

```edgeql
insert Country {
  name := 'Hungary'
};
insert Country {
  name := 'Romania'
};
```

顺便说一下，你可能注意到了 `Place` 中的 `important_places` 仍然是一个字符串数组 `array<str>`，也许改为 `multi` 链接可能会更好？确实如此，但是在本教程中我们最终并未使用这个属性，所以我们就简单地将其作为数组保留在这个结构中，不做过多调整了。如果这是一个正式的架构设计，它可能最终会变成一个“多链接”`multi` 链接，或者因为我们并不需要它而被删除掉。

## 捕获 select 表达式

现在，我们准备为“德古拉”创建一条相应的数据。首先，我们已将 `Person` 中的 `places_visited` 由 `City` 改为 `Place` 使得它有更多可能性，比如：伦敦（London），比斯特里察（Bistritz），匈牙利（Hungary）等等。并且，我们目前只是知道德古拉一直待在罗马尼亚（Romania），所以当我们想“选择罗马尼亚”赋值给德古拉的 `place_visited` 时，我们可以通过对 `Place` 做一个快速的 `filter` 从而选出罗马尼亚。这样做时，我们需要将 `select` 语句放在一个 `()` 圆括号内，该括号是捕获 `select` 查询结果所必需的，然后我们可以使用该查询结果来做一些事情。换句话说，括号界定了（设置边界）`select` 查询，从而使其结果可以作为一个整体被使用。即，EdgeDB 将在括号内进行操作，然后将完成的结果提供给 `places_visited`。

```edgeql
insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
  # .places_visited is the result of this select query.
};
```

执行结果是：`{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`

`uuid` 是来自服务器的回复，显示我们刚刚创建了哪个对象并表明创建成功。

现在，让我们来检查一下 `places_visited` 是否可以工作。目前我们只是有一个 `Vampire` 的对象，对它进行 `select`：

```edgeql
select Vampire {
  places_visited: {
    name
  }
};
```

执行结果是：`{default::Vampire {places_visited: {default::Country {name: 'Romania'}}}}`

完美。

## 添加约束

现在，让我们再回过头关注一下 `age`。对于 `Vampire` 类型，这很容易，因为他们可以永生。但是现在我们也想给不可能永生的人类 `PC` 和 `NPC` 一个 `age`，但我们并不会期待他们能活 32767 年。对于这一点，我们在这里会学习如何使用“约束”（constrait，一个限制）。首先，我们会为人类的 `age` 创建一个名为 `HumanAge` 的新类型，而并不是简单的使用 `int16`。然后，我们会对在定义该类型时使用 `constraint` 以及函数 {ref}`functions <docs:ref_datamodel_constraints>` 中的 `max_value()`。 

下面是 `max_value()` 的签名：

`std::max_value(max: anytype)`

`anytype` 部分比较有趣，这表示出数字以外它也可以处理像字符串这样的类型。例如，如果使用了约束 `max_value('B')` 就表示不允许使用“B”以外的字符，如“C”、“D”等。

现在让我们回到我们对 `HumanAge` 的约束，即 120。`HumanAge` 类型定义如下所示：

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

请记住，它是一个标量类型，因为它只能有一个值。然后我们将它添加到 `NPC` 类型。

```sdl
type NPC extending Person {
  age: HumanAge;
}
```

`HumanAge` 是我们自己的类型，有自己的名字，它是一个不能大于 120 的 `int16`。所以如果我们像下方一样进行插入，将无法工作：

```edgeql
insert NPC {
    name := 'The innkeeper',
    age := 130
};
```

这里会给出一个错误提示：`ERROR: ConstraintViolationError: Maximum allowed value for HumanAge is 120.`

现在，如果我们将 `age := 130` 更改为 `age := 30`，我们会得到表明插入成功的消息：`{Object {id: 72884afc-f2b1-11ea-9f40-97b378dbf5f8}}`。总之，现在没有 NPC（非玩家角色 Non-Player Character）可以超过 120 岁。

## 删除对象

在 EdgeDB 里删除是十分容易的：只需要使用关键字 `delete`。类似于 `select`，当你键入 `delete` 和类型后，默认情况下会删除该类型的所有。和 `select` 一样，如果你使用 `filter` 那么它只会删除过滤器匹配到的那些数据。

这种与 `select` 的相似性可能会让你感到紧张，因为如果你键入类似 `select City` 的内容，它将选择 `City` 的所有内容。`delete` 是相同的：`delete City` 会删除 `City` 类型的每一个对象。这就是为什么我们在删除任何内容前都需要再三考虑。

所以让我们来试一下。还记得我们创建的匈牙利（Hungary）和罗马尼亚（Romania）两个 `Country` 类型的对象吗？现在让我们来删除它们：

```edgeql
delete Country;
```

此时，我们会收到错误，它告诉我们无法删除 `Country`，因为 `Country` 类型的对象仍然在被某个 `Person` 对象的 `places_visited` 所引用。

```
ERROR: ConstraintViolationError: deletion of default::Country (7f3c611c-ff43-11eb-af59-dfe5a152a5cb) is prohibited by link target policy
  Detail: Object is still referenced in link places_visited of default::Person (7f5b25ac-ff43-11eb-af59-3f8e155c6686).
```

这正是正待在罗马尼亚的德古拉伯爵带来的麻烦。所以我们在这里先暂时删除它：

```edgeql
delete Vampire;
```

就像插入一样，EdgeDB 将返回我们现在正在删除的所有对象的 id：`{default::Vampire {id: 7f5b25ac-ff43-11eb-af59-3f8e155c6686}}`。现在，我们可以再试一下删除 `Country`：

```edgeql
delete Country;
```

我们得到确认，即两个 `Country` 对象已被删除：

```
{
    default::Country {id: 7f2da5e6-ff43-11eb-af59-33db995c2682}, default::Country {id: 7f3c611c-ff43-11eb-af59-dfe5a152a5cb},
}
```

现在，让我们把它们再重新插入回去，并试一下带过滤的删除：

```edgeql
delete Country filter .name ilike '%States%';
```

没有任何匹配项，因此输出为 `{}`——我们没有删除任何内容。让我们再试一次：

```edgeql
delete Country filter .name ilike '%ania%';
```

这次我们得到 `{default::Country {id: 7f3c611c-ff43-11eb-af59-dfe5a152a5cb}}`，即 Romania。现在只剩下 Hungary 了。如果我们想查看我们到底删除了什么怎么办？没问题——只需将 `delete` 放在圆括号内，然后使用 `select` 即可。现在，让我们再次删除所有 `Country` 对象，但这次我们将 `select` 删除结果：

```edgeql
select (delete Country) {
  name
};
```

输出结果是：`{default::Country {name: 'Hungary'}}`，这说明我们删除了 Hungary。如果现在我们执行 `select Country`，我们将得到一个空集 `{}`，这说明我们确实把它们都删了。

（有趣的是：在 EdgeDB 里，`delete` 语句实际上是 `delete (select ...)` 的语法糖 {ref}`syntactic sugar <docs:ref_eql_statements_delete>`。另外，在下一章中，你将通过使用 `select` 学习关键字 `limit`。当你学习使用它时，请记住你也同样可以将其应用到 `delete`。）

> 语法糖（英语：Syntactic sugar）是由英国计算机科学家彼得·兰丁发明的一个术语，指计算机语言中添加的某种语法，这种语法对语言的功能没有影响，但是更方便程序员使用。语法糖让程序更加简洁，有更高的可读性。

最后，让我们再次插入匈牙利（Hungary）和罗马尼亚（Romania）以供后续使用。

[→ 点击这里查看到第 3 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 此查询试图显示每个 `NPC` 的 `name`，并给每个 `NPC` 加上所有 `City`，但执行后出现了错误。请问它缺少了什么？

   ```edgeql
   select NPC {
     name,
     cities := select City.name
   };
   ```

2. 如果 `City` 类型需要一个名为 `population` 的 `required` property 属性，它会是什么样子？“population”会是什么类型？
3. 此查询出于某种原因想要显示两次 `name`，但出现了错误。你能想出办法解决吗？

   ```edgeql
   select Person {
     name,
     name
   };
   ```

   （提示：问题是名称 `name` 被使用了两次）

4. 有人一直在尝试给某个角色赋予负数年龄，你可以想出一种阻止这种情况的约束吗？

   （提示：当前约束是`max_value(120);`）

5. 你能插入一个 HumanAge 类型吗？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森：“德古拉伯爵对历史太了解了！我很庆幸我来了。”_
