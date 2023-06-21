---
tags: Type Annotations, Backlinks
---

# 第十四章 - 一线希望

> 终于我们还是迎来了一些好消息：乔纳森·哈克（Jonathan Harker）还活着。他逃离了德古拉城堡（Castle Dracula），在 8 月份找到了前往布达佩斯（Budapest）的路，随后进入了一家医院，医院给米娜（Mina）发了一封信，告诉米娜：“乔纳森受到了一些可怕的刺激，不停地说着狼、毒和血，鬼和恶魔。” 米娜立即乘坐火车前往乔纳森所在的康复医院，并接他一起乘坐火车回到了英格兰（England）。之后，他们在埃克塞特（Exeter）举行了婚礼。米娜从埃克塞特寄了一封信给露西，告诉了她这个好消息……但是太迟了，露西再也没能打开它。与此同时，上一章提到的男士们按计划到访了墓地，看到了吸血鬼露西在四处走动。亚瑟（Arthur）看到后，终于相信了范海辛（Van Helsing），其他人也是如此。现在他们相信了吸血鬼是真实存在的，并在设法摧毁了露西。亚瑟十分伤心，但也为露西将不再被迫成为一只吸血鬼，且很快可以平静地离开人世而感到开心。

这里，我们有了一个名为“Exeter（埃克塞特）”的新城市，添加它很容易：

```edgeql
insert City {
  name := 'Exeter',
  population := 40000
};
```

40000 是当时埃克塞特的人口，且它没有一个与当时书中名称不同的 `modern_name`。

## 添加类型注解并使用 @

既然我们已经知道了如何进行内省查询，我们就可以开始给我们的类型添加 `annotations`（注解）了。注释是类型定义中的一个字符串，它为我们提供有关该类型的说明。默认情况下，注释可以使用标题 `title` 或 `description`。

假设在我们的游戏中，一个 `City` 至少需要有 50 座建筑物才能被称为城市。现在，让我们运用 `description` 对此加以说明：

```sdl
type City extending Place {
  annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
  population: int64;
}
```

然后，我们可以对其进行 `introspect` 查询。我们从上一章就知道如何做到这一点——即只需在各处添加上 `: {name}` 则可获取其内部细节：

```edgeql
select (introspect City) {
  name,
  properties: {name},
  annotations: {name}
};
```

哦，似乎还不够：

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {schema::Annotation {name: 'std::description'}},
  },
}
```

当然：`annotations: {name}` 部分返回的是 _类型_ 的名称，即 `std::description`。换句话说，它是一个链接，链接的目标只是告诉我们所使用的注释的类型，但我们实际要找的是其中的“值”。

这就是 `@` 的用武之地。为了获得里面的值，我们要写入：`@value`。`@` 用于直接访问内部的值（字符串），而不仅仅是类型名称。让我们再试一次：

```edgeql
select (introspect City) {
  name,
  properties: {name},
  annotations: {
    name,
    @value
  }
};
```

现在我们看到实际的注解了：

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'A place with 50 or more buildings. Anything else is an OtherPlace',
      },
    },
  },
}
```

如果我们想要一个区别于 `title` 和 `description` 的、具有不同名称的注释该怎么办？这很容易，只需要在架构中声明 `abstract annotation` 并为其命名即可。比如，如果我们想添加一个警告，我们可以这样做：

```sdl
abstract annotation warning;
```

假设 `Castle` 类型将替代 `OtherPlace` 类型用于城堡，甚至是城堡镇，我们则可以用上面新定义的警告注释为 `OtherPlace` 提供更多说明：

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
}
```

现在，让我们仅对其名称和注释进行内省查询：

```edgeql
select (introspect OtherPlace) {
  name,
  annotations: {name, @value}
};
```

结果是：

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {name: 'std::description', @value: 'A place with under 50 buildings - hamlets, small villages, etc.'},
      schema::Annotation {name: 'default::warning', @value: 'Castles and castle towns do not count! Use the Castle type for that'},
    },
  },
}
```

## 更多关于日期

如今，已经有很多角色陆续死去了，所以让我们现在来考虑一下这个问题。我们可以运用 `cal::local_date` 来判断谁还活着，谁已经死了。首先，我们先来看一下到目前为止我们已经拥有了多少个 `Person` 对象。为此，我们可以使用 `select count(Person)` 轻松得到结果是 `{24}`。

这里再引出一个名为 {eql:func}`docs:std::enumerate` 的函数，该函数接受任何集合，并会返回一组形式为 `(index, element)` 的元组，每个元组项包含从零开始计数的索引以及输入集合中对应的元素。我们将用它与 `count()` 函数的结果进行比较，以确保我们得到的数字是正确的。

下面是运用了 `enumerate()` 的一个简单示例：

```edgeql
with three_things := {'first', 'second', 'third'},
select enumerate(three_things);
```

输出是：

```
{(0, 'first'), (1, 'second'), (2, 'third')}
```

现在，让我们来执行 `select enumerate(Person.name);`：如果最后一个索引是 23，则验证了我们得到了 24 个结果：

```
{
  (0, 'Jonathan Harker'),
  (1, 'Renfield'),
  (2, 'The innkeeper'),
  (3, 'Mina Murray'),
  (4, 'John Seward'),
  (5, 'Quincey Morris'),
  (6, 'Arthur Holmwood'),
  (7, 'Abraham Van Helsing'),
  (8, 'Lucy Westenra'),
  (9, 'Vampire Woman 1'),
  (10, 'Vampire Woman 2'),
  (11, 'Vampire Woman 3'),
  (12, 'Lucy'),
  (13, 'Count Dracula'),
  (14, 'The Captain'),
  (15, 'Petrofsky'),
  (16, 'The First Mate'),
  (17, 'The Cook'),
  (18, 'Emil Sinclair'),
}
```

但只有 19 个？哦，没错：因为我们有 5 个 `Crewman` 对象没有名称，所以它们不会出现。那我们该如何在查询中获取它们呢？我们当然可以像下面这样做：

```edgeql
with
  a := array_agg((select enumerate(Person.name))),
  b := array_agg((select enumerate(Crewman.number))),
select (a, b);
```

（`array_agg()` 是为了避免发生集合乘以集合的情况，如果对此你记得不太清楚了，可以重新回顾一下第 12 章）

但执行后的结果并不令人满意：

```
{
  (
    [
      (0, 'Jonathan Harker'),
      (1, 'Renfield'),
      (2, 'The innkeeper'),
      (3, 'Mina Murray'),
      # snip
      (16, 'The First Mate'),
      (17, 'The Cook'),
      (18, 'Emil Sinclair'),
    ],
    [(0, 1), (1, 2), (2, 3), (3, 4), (4, 5)],
  ),
}
```

`Crewman` 类型在结果中的名字只是数字，这看起来不太友好。所以，让我们放弃像上面一样花哨的查询，先用基于数字的名称对 `Crewman` 的 `name` 进行更新：

```edgeql
update Crewman
set {
  name := 'Crewman ' ++ <str>.number
};
```

现在，每个人都有名字了（此时再执行 `select enumerate(Person.name);`，结果中的最后一个索引将为 23），让我们再来看看他们是否都还在世。逻辑很简单：我们输入一个 `cal::local_date`，如果它大于 `last_appearance` 的日期，那么表明这个角色已经死了。

```edgeql
with p := (select Person),
     date := <cal::local_date>'1893-08-16',
select (p.name, p.last_appearance, 'Dead on ' ++ <str>date ++ '? ' ++ <str>(date > p.last_appearance));
```

这是输出（只有 6 个结果，因为其他对象尚未设置过 `last_appearance`）：

```
{
  ('Lucy Westenra', <cal::local_date>'1893-09-20', 'Dead on 1893-08-16? false'),
  ('Crewman 1', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 2', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 3', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 4', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
  ('Crewman 5', <cal::local_date>'1893-07-16', 'Dead on 1893-08-16? true'),
}
```

如果这个判断我们用得足够多的话，我们当然可以考虑把它变成一个函数。

## 反向链接

最后，让我们看看如何反向跟踪链接，这是 EdgeDB 最强大和最有用的功能之一。学习如何使用它可能需要费些力气，但非常值得。

首先，我们可以像下面这样获取德古拉伯爵的所有 `slave` 的名称：

```edgeql
select Vampire {
  name,
  slaves: {
    name
  }
};
```

输出结果如下所示：

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

但是如果我们想做相反的事情呢？即用 `select MinorVampire` 开头，并访问与其链接的 `Vampire` 对象，该怎么做呢？毕竟现在的 `MinorVampire` 只能调用属于 `MinorVampire` 类型和抽象类型 `Person` 的属性：

```edgeql
select MinorVampire {
  name,
  # master... 怎么找呢?
  # 在MinorVampire没有到Vampire的链接...
}
```

既然上面没有 `master: Vampire`，我们该如何倒退查看链接到它的 `Vampire` 类型呢？

这就要靠反向链接了，我们使用 `.<` 代替 `.` 并指定我们要查找的类型：`[is Vampire]`。

首先，让我们抛开对 `MinorVampire` 的查询，看看`.<` 是如何工作的。这有一个例子：

```edgeql
select MinorVampire.<slaves[is Vampire] {
  name,
  age
};
```

这并不是在选择 `MinorVampire` 的 `slaves`，它是在选择 `Vampire`，且需要满足其 `.slaves` 链接有指向的 `MinorVampire` 对象。

你可以将 `MinorVampire.<slaves[is Vampire] {name, age}` 视为：选择“拥有 MinorVampire 作为奴隶”的 Vampire 的名称和年龄”。

这里是输出：

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```

这与 `select Vampire: {name, age}` 的输出结果相同，但它在我们之前想要访问多种类型的查询中变得非常有用。现在我们可以选择所有 `MinorVampire` 对象以及它们的主人了：

```edgeql
select MinorVampire {
  name,
  master := .<slaves[is Vampire] {name},
};
```

你可以将 `.<slaves[is Vampire] {name}` 解读为“通过 `.slaves` 链接到 `MinorVampire` 的 `Vampire` 的名称”。 

这里是输出：

```
{
  default::MinorVampire {name: 'Vampire Woman 1', master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Vampire Woman 2', master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Vampire Woman 3', master: {default::Vampire {name: 'Count Dracula'}}},
  default::MinorVampire {name: 'Lucy', master: {default::Vampire {name: 'Count Dracula'}}},
}
```

[→ 点击这里查看到第 14 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何仅显示所有 `Person` 对象的编号？比如，如果有 20 个，则显示 `1, 2, 3..., 18, 19, 20`。

2. 使用一个可计算的反向链接，如何显示 1）所有名称中带有 `o` 的 `Place` 的对象（及他们的名称）；2）访问过这些地方的人物的名字？

3. 使用一个可计算的反向链接，如何显示所有后来成为了 `MinorVampire` 的 `Person` 对象？

   提示：别忘了，`MinorVampire` 有一个指向自己（成为吸血鬼之前）前身的链接。

4. 如何给 `MinorVampire` 类型一个名为 `note` 的注解，并添加说明 `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`？

5. 如何在查询中看到 `MinorVampire` 的 `note` 注释？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候报仇了。_
