---
tags: Filtering On Insert, Json
---

# 第六章 - 还是逃不掉

> 三个女吸血鬼就在乔纳森的身边，但他无法动弹。突然，德古拉跑进了房间，命令这些女人离开：“之后你们可以拥有他，但今晚不行！”。女吸血鬼们随后便离开了。乔纳森醒来时躺在他的床上，感觉就像做了一场噩梦……但当他看到有人叠好了他的衣服，他就知道这不会只是个梦。因为第二天会有一些来自斯洛伐克的游客参观城堡，所以乔纳森想到了个主意。他写了两封信，一封给米娜，一封给他的老板。他给了来访者一些钱，请求他们为他寄信。但还是被德古拉发现了。德古拉很生气并在乔纳森面前烧毁了信件，并警告他不许再犯。乔纳森仍然被困在城堡里，但德古拉也清楚乔纳森仍然在想办法试图逃跑。

## 插入时使用过滤

本章中没有太多关于类型的新内容，所以让我们来看看如何改进我们现有的架构（schema）。目前，我们对乔纳森·哈克（Jonathan Harker）的插入语句仍然是这样的：

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

当我们只有城市时，问题不大。但现在我们增加了 `Place` 和 `Country` 两个新类型，情况变得复杂了一些。首先，我们先来插入两个新的 `Country` 对象以增加多样性：

```edgeql
insert Country {
  name := 'France'
};
insert Country {
  name := 'Slovakia'
};
```

（在第 9 章中，我们将学习如何只用一个 `insert` 来做到同时插入多个对象！）

现在，我们接着为既不是城市也不是国家的地方创建二个名为 `Castle` 和 `OtherPlace` 的新类型。这很容易做到：

```sdl
type Castle extending Place;
type OtherPlace extending Place;
```

然后，我们要更新已插入 Count Dracula 的 `Castle`：

```edgeql
update Castle filter .name = 'Castle Dracula'
  set {
    doors := [6, 9, 10]
  };
```

然后 `OtherPlace` 可以为我们存储大量来自 `Place` 类型但不属于 `City` 类型的地点。

回到乔纳森：在我们的数据库中，他去过四个城市、一个国家和一个 `Castle`……但他没有去过斯洛伐克（Slovakia）或法国（France），所以我们不能直接用 `places_visited := select Place` 对其进行插入。但我们可以根据他访问过的地方的名称对 `Place` 进行过滤。像这样：

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := (
    select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'}
  )
};
```

你会留意到我们只是使用了 `{}` 并在集合中写入了这些地方的名称，我们不需要使用带有 `[]` 的数组来执行此操作。（顺便说一下，这称为 set 构造函数 {ref}`set constructor <docs:ref_eql_set_constructor>`）

现在，如果乔纳森逃离了德古拉城堡并来到了一个新地方，该怎么办？让我们假设他成功逃跑了并逃到了斯洛伐克（Slovakia）。当然，我们可以修改他的 `insert`，在造访点的名字集合里加上 `'Slovakia'`。但是我们如何才能做一次快速的更新呢？对此，我们有关键字 `update` 和 `set`。`update` 要更新的类型，`set` 我们要修改的部分。像这样：

```edgeql
update NPC
filter .name = 'Jonathan Harker'
set {
  places_visited += (select Place filter .name = 'Slovakia')
};
```

如果语句执行成功了，EdgeDB 将返回更新成功的对象的 ID。在上面的例子里，仅仅会返回一个 ID（因为只有一个对象被更新）：

```
{default::NPC {id: ca4e21c8-014f-11ec-9658-7f88bf45dae6}}
```

如果我们写了类似 `filter .name = 'SLLLlovakia'` 的过滤，那么 EdgeDB 会返回 `{}`，让我们知道没有匹配到任何结果。或者更准确地说：顶级对象在 `filter .name = 'Jonathan Harker'` 上得到匹配，但 `places_visited` 并没有得到更新，因为 `filter` 里没有获得任何可匹配的对象。

由于乔纳森（Jonathan）实际上还没有访问过斯洛伐克（Slovakia），我们现在再将它删除，你可以用 `-=` 代替 `+=`，并使用相同的 `update` 语法来删除它。

现在我们了解了在 `set` 之后可以使用的 {ref}`三个运算符 <docs:ref_eql_statements_update>`：`:=`、`+=` 和`-=`。 

接下来，让我们来做另一个更新。还记得这个吗？

```edgeql
select Person {
  name,
  lover
} filter .name = 'Jonathan Harker';
```

米娜·默里（Mina Murray）的 `lover` 是乔纳森·哈克（Jonathan Harker），但因为我们先插入了乔纳森，所以乔纳森却一直没有 `lover`。现在，我们可以更新它了：

```edgeql
update Person filter .name = 'Jonathan Harker'
set {
  lover := assert_single(
    (select detached Person filter .name = 'Mina Murray')
  )
};
```

这里我们需要使用 `detached Person`，与我们在之前的章节中用 `insert NPC` 创建 Mina Murray 并指定她爱人时使用了 `detached` 的理由一样。我们要在一个 `Person` 上使用 `update`，同时还需要 `select` 另一个 `Person` 作为其爱人。现在，Jonathan 的 `lover` 链接终于显示了 Mina 而不再是空的 `{}`。

当然，如果你在没有 `filter` 的情况下使用了 `update`，它将对所有对象进行相同的更改。例如，下面的语句会在数据库里把所有 `Place` 更新进每一个 `Person` 对象的 `places_visited` 属性中：

```edgeql
update Person
set {
  places_visited := Place
};
```

## 级联运算符 ++

还有一个运算符是 `++`，它执行级联（即，连接在一起）而不是做“相加”。

你可以像这样：`select 'My name is ' ++ 'Jonathan Harker';` 做简单的操作，从而得到结果 `{'My name is Jonathan Harker'}`。或者你可以做更复杂的级联，只要你不断将字符串连接到字符串：

```edgeql
select 'A character from the book: ' ++ (select NPC.name) ++ ', who is not ' ++ (select Vampire.name);
```

输出为：

```
{
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
  'A character from the book: Jonathan Harker, who is not Count Dracula',
}
```

（这个级联运算符也适用于数组，它可以将被操作的数组合并放入到一个数组中。所以执行 `select ['I', 'am'] ++ ['Jonathan', 'Harker'];` 的结果是 `{['I', 'am', 'Jonathan', 'Harker']}`。）

现在，让我们再来更改一下 `Vampire` 类型，使其拥有一个新链接，并指向其所掌控的 `MinorVampire`。你应该还记得德古拉伯爵是游戏中唯一一个真正的吸血鬼，而其他吸血鬼都是 `MinorVampire` 类型。这意味着我们需要一个 `multi` 链接：

```sdl
type Vampire extending Person {
  age: int16;
  multi slaves: MinorVampire;
}
```

然后，我们便可以在插入德古拉伯爵（Count Dracula）信息的同时 `insert` 相关的 `MinorVampire` 对象。但首先让我们先从 `MinorVampire` 中删除 `required master`，因为我们不希望两个对象相互链接。原因有二：

- 会使事情变得复杂。假设我们声明的 `Vampire` 里包含一个指向 `MinorVampire` 的 `slaves` 链接，在我们插入 `Vampire` 时，如果我们还没有创建对应的 `MinorVampire`，那么它将是空 `{}`，则我们将不得不在创建 `MinorVampire` 后，再对 `Vampire` 进行更新；如果我们先创建的是 `MinorVampire`，且它有一个指向 `Vampire` 的 `master` 链接，而我们还没有创建 `Vampire`，那么这个 `master` 将不存在，尤其当它是个 `required` link 时，我们必须提供一个 `master`。
- 如果这两种类型相互链接，我们将无法在需要时删除它们。删除时的错误提示如下所示：

```
db> delete MinorVampire;
ERROR: ConstraintViolationError: deletion of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114) is prohibited by link target policy
  Detail: Object is still referenced in link slave of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6).
db> delete Vampire;
ERROR: ConstraintViolationError: deletion of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6) is prohibited by link target policy
  Detail: Object is still referenced in link master of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114).
```

因此，我们先简单地将 `MinorVampire` 改为没有 `master` 链接的 `Person` 扩展类型：

```sdl
type MinorVampire extending Person;
```

然后，我们在创建德古拉伯爵时，可以一并创建德古拉控制的小鬼们，如下所示：

```edgeql
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (insert MinorVampire {
      name := 'Vampire Woman 1',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 2',
    }),
    (insert MinorVampire {
      name := 'Vampire Woman 3',
    }),
  }
};
```

请注意两件事：（1）我们在 `slaves` 后面使用了 `{}`，并把所有要插入的 `insert` 都放到了这个集合里；（2）每一个 `insert` 都放在了圆括号 `()` 里去捕捉插入。

现在我们不必像以前一样，先插入 `MinorVampire` 类型，然后再通过过滤并更新给 `Vampire`：我们可以将她们与德古拉（Dracula）放在一起进行插入并完成关系的建立。

然后当我们 `select Vampire` 时：

```edgeql
select Vampire {
  name,
  slaves: {name}
};
```

我们可以由此得到想要的输出结果，将德古拉（Dracula）和女吸血鬼们一并显示出来：

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Vampire Woman 1'},
      default::MinorVampire {name: 'Vampire Woman 2'},
      default::MinorVampire {name: 'Vampire Woman 3'},
    },
  },
}
```

这可能会使你好奇：如果我们就是想建立双向链接怎么办？实际上这里确实有一种非常方便的方法（称为 **向后链接**），但我们要到第 14 章和第 15 章才会看它。如果你真的很好奇，你可以直接跳到后面的章节，但在那之前我们还有很多东西要学。

## 用 \<json> 生成 JSON

如果我们想要输出 JSON 格式，该怎么做？再简单不过了：使用 `<json>` 进行转换即可。EdgeDB 中的任何类型（`bytes` 除外）都可以轻松地转换为 JSON：

```edgeql
select <json>Vampire {
  # <json> is the only difference from the `select` above
  name,
  slaves: {name}
};
```

这样做我将会得到转为 JSON 的结果。然而，REPL（read-eval-print loop，读取-求值-输出循环）在默认情况下显示出的内容看起来更像是这样：

```
{
  "{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Woman 1\"}, {\"name\": \"Woman 2\"}, {\"name\": \"Woman 3\"}]}",
}
```

现在让我们一起过一下这个结果。最外层的花括号只是告诉你里面是查询返回的一个或多个结果，而实际的结果是一个 JSON 字符串。因为 JSON 部分在字符串中，所以所有的 `"` 都需要转义，写为 `\"`。

要让 REPL 以更友好的格式显示 JSON，只需输入 `\set output-format json-pretty`。这样结果看起来会是我们更加熟悉的样式：

```json
{
  "name": "Count Dracula",
  "slaves": [{"name": "Woman 1"}, {"name": "Woman 2"}, {"name": "Woman 3"}]
}
```

如果你要回复默认的格式，输入 `\set output-format default` 即可。

## 从 JSON 转换回来

那么反过来呢，就是把 JSON 转回成 EdgeDB 的类型呢？这也是可行的，但要当心 JSON 数据的类型，因为 EdgeDB 讲究转换的对称性：转成 JSON 前是什么类型，转回来还应该是什么类型。比如，《德古拉》里出现的第一个日期字符串是 `'18930503'`，如果我们尝试将它的 JSON 值转换为一个 `int64`，它将无法正常工作：

```edgeql
select <int64><json>'18930503';
```

问题出在从 JSON 字符串到 EdgeDB `int64` 的转换，错误提示为：`ERROR: InvalidValueError: expected json number or null; got json string`。也就是说，EdgeDB `str` 转换为 JSON 字符串后，又试图转换为 EdgeDB `int64`，这是不可行的。（除非是 EdgeDB 数字类型转换为 JSON 数字，才可再转换回 EdgeDB 数字类型。）因此，这里为了保持对称，你需要先将 JSON 字符串转换为 EdgeDB 的 `str`，然后再转换为 `int64`：

```edgeql
select <int64><str><json>'18930503';
```

现在它正常工作了：我们可以得到 `{18930503}`，过程是 EdgeDB `str`，变成了 JSON 字符串，然后又回到 EdgeDB `str`，最后被转换为 `int64`。

接下来，我们再来看下面的列子，即把《德古拉》里出现的第一个日期字符串转换成 JSON 字符串后再转换回成 `cal::local_date` 类型：

```edgeql
select <cal::local_date><json>'18930503';
```

你会发现它是可以正确执行的，结果是 `{<cal::local_date>'1893-05-03'}`。因为 `<json>` 将 `'18930503'` 转换为了 JSON 字符串，且 `cal::local_date` 可以接收 JSON 字符串以进行创建。换句话说，当你对 EdgeDB 的 `cal::local_date` 进行 JSON 转换时，你将得到一个 JSON 字符串，因此反之，你固然可以将符合日期格式的 JSON 字符串直接转换回 `cal::local_date`。

关于 JSON 的文档 {ref}`documentation on JSON <docs:ref_std_json>` 解释了哪些 JSON 类型可以转换为哪些 EdgeDB 类型，列出了可以处理 JSON 值的函数，如果在你的应用中需要对 JSON 进行大量转换，可以将其添加至书签。

[→ 点击这里查看到第 6 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 下面这个选择是不完整的。如何修改它从而使它能打印出“Pleased to meet you, I'm ”以及 NPC 的名字？

   ```edgeql
   select NPC {
     name,
     greeting := ## Put the rest here
   };
   ```

2. 如果米娜要去德古拉城堡参观，你会如何更新米娜（Mina）的 `places_visited`，让它也包括罗马尼亚？

3. 如何显示出名称（name）里包含 `{'W', 'J', 'C'}` 中任何一个大写字母的所有 `Person` 对象？

   提示：使用 `with` 和级联 `++`（concatenation）操作。 

4. 如何用 JSON 显示和上一题相同的查询？

5. 如何将“ the Greate”添加到每个 Person 类型的对象中？

   此外，使用字符串索引来撤消此操作的快速方法是什么？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森爬上城墙进入德古拉的房间。_
