---
tags: Introspection, Type Union Operator
---

# 第十三章 - 认识新的露西

> 这一次，人们对露西（Lucy）的挽救已经来不及了，她快要死了。突然，她睁开了眼睛，但看起来十分诡异。她看着亚瑟（Arthur）说：“亚瑟！亲爱的，我很高兴你来看我！快亲吻我！” 在亚瑟正要去吻她时，范海辛（Van Helsing）抓着他说道：“你怎么敢！” 范海辛很清楚，说话的已经不是露西了，而是她身体里面的吸血鬼。露西已经死了，范海辛把一个金色的十字架放在她的嘴唇上以阻止她移动（十字架对吸血鬼有这种力量）。不幸的是，一个护士趁没人看见的时候偷了十字架去卖。几天后，有消息传出称一个女性正在偷窃和啃咬孩子——这一定是吸血鬼露西。报纸称它为“Bloofer Lady”，这是因为年幼的孩子们想叫她“Beautiful Lady”（美丽的女士），但又不能正确地念出 _beautiful_。现在范海辛把吸血鬼的真相告诉了大家，但亚瑟仍无法相信，并且非常气愤他竟用如此疯狂的事情诋毁自己的妻子。范海辛说：“好吧，你不相信我。那让我们今晚一起去墓地看看会发生什么。也许到时候你就相信了。” 

看起来原本是 `NPC` 的露西已经变成 `MinorVampire`（小吸血鬼）了。我们该如何在数据库里展示这一点呢？首先，让我们先重新过一遍这些类型。

当前的 `MinorVampire` 还没什么特别的属性，只是一个扩展自 `Person` 的类型：

```sdl
type MinorVampire extending Person;
```

接着，按小说的描述，露西似乎带来了一个新的人类“类型”。旧的露西已经消失了，新的露西是一个名为德古拉伯爵的 `Vampire`（吸血鬼）的 `slaves`（奴隶）。

因此，与其尝试更改 `NPC` 类型，不如只是为 `MinorVampire` 增加一个指向 `Person` 的可选链接：

```sdl
type MinorVampire extending Person {
  former_self: Person;
}
```

之所以设置为可选的，是因为我们并不一定知道所有 `MinorVampire` 在成为吸血鬼之前都是谁。例如，我们对德古拉控制的那三个女吸血鬼的身世就一无所知，因此我们无法为她们制作 `NPC` 类型来尝试链接到她们的 `former_self` 属性。

另一个对 `MinorVampire` 与其前身 `NPC` 可以做的（非正式的）关联是给 `NPC` 的 `last_appearance` 和 `MinorVampire` 的 `first_appearance` 设置相同的日期。首先，我们先更新 `NPC` 露西的 `last_appearance`：

```edgeql
update Person filter .name = 'Lucy Westenra'
set {
  last_appearance := cal::to_local_date(1893, 9, 20)
};
```

然后我们可以将露西（Lucy）添加到德古拉（Dracula）的 `insert` 中。（如果前面章节里的操作你都有执行，那么现在只需先执行 `delete Vampire;` 和 `delete MinorVampire;`，然后我们就可以通过插入 `Vampire` 重新练习 `insert` 了）

注意下面语句中的第一行，我们创建了一个名为 `lucy` 的变量。然后我们使用它来引入所有数据，为露西创建 `MinorVampire`，这比插入时反复书写 `select` 语句录入所有信息要高效得多。这里还需要包括露西的力量：我们给它额外加 5，因为吸血鬼会更加强壮些。

下面是具体的插入语句：

```edgeql
with lucy := assert_single(
    (select Person filter .name = 'Lucy Westenra')
)
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
    (insert MinorVampire {
      name := 'Lucy',
      former_self := lucy,
      first_appearance := lucy.last_appearance,
      strength := lucy.strength + 5,
    }),
  },
  places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'})
};
```

多亏了 `former_self` 链接，我们很容易找到所有来自 `Person` 对象的小吸血鬼。即只需用 `exists .former_self` 进行过滤：

```edgeql
select MinorVampire {
  name,
  strength,
  first_appearance,
} filter exists .former_self;
```

输出结果：

```
{
  default::MinorVampire {
    name: 'Lucy',
    strength: 5,
    first_appearance: <cal::local_date>'1893-09-20',
  },
}
```

使用其他的过滤方式也可以，比如使用 `filter .name in Person.name and .first_appearance in Person.last_appearance;` 也是可以的，但检查链接是否 `exists`（存在）最简单（也最保险）。我们还可以用 `cal::local_datetime` 替代 `cal::local_date` 作为时间属性的类型，以获得更精确的时间，但是我们现在还用不着。

## 类型联合运算符：|

另一个与类型相关的运算符是 `|`，用于联合类型（类似于 `or`）。例如，下面的查询拉出了所有 `Person` 类型以判断露西所属类型是否在其中，并返回“真”：

```
select (select Person filter .name like 'Lucy%') is NPC | MinorVampire | Vampire;
```

意思是如果选择的 `Person` 对象是 `NPC` 或 `MinorVampire` 或 `Vampire` 其中的任一个类型，则返回“真”。由于露西的 `NPC` 身份和露西的 `MinorVampire` 身份可以分别匹配到这三种类型中的一种，所以返回值是 `{true, true}`。

还有一件很酷的事情是：你也可以在链接中使用类型联合运算符。例如，让我们临时假设游戏中还有其他 `Vampire` 对象，其中一个 `Vampire` 非常强大，可以控制另一个。但在我们当前的架构里，`Vampire` 只能控制 `MinorVampire`，而不能控制 `Vampire`，即：

```
type Vampire extending Person {
  multi slaves: MinorVampire;
}
```

为了体现这个临时的变更，你可以使用 `|` 并添加另一种类型：

```
type Vampire extending Person {
  multi slaves: MinorVampire | Vampire;
}
```

然而，我们的数据库中实际上只有德古拉伯爵（Count Dracula）是 `Vampire`，因此我们并不打算真的以上面这种方式更改我们的架构，这里只是为了示意你还可以如何做，但请记住 `|` 运算符，也许有天你会需要它。

## 链接目标删除策略

我们在这里决定为露西保留旧的 `NPC` 对象，因为作为 `NPC` 的露西将在游戏中待到 1893 年 9 月 20 日，也许会有 `PC` 类型的对象会与她有所互动。但这可能会让你想了解关于链接的删除。即如果我们在她成为 `MinorVampire` 时想删除她的旧类型对象，该怎样？或者更现实地说，当吸血鬼死时，想要同时删除所有链接到 `Vampire` 的 `MinorVampire` 对象，该怎么办？我们不会在我们的游戏中真的这样做，但你确实可以使用 `on target delete` 来实现。`on target delete` 的意思是“当链接目标被删除时”，执行其后面（在 `{}` 里）命令。为此，我们有 {ref}`四个选项 <docs:ref_datamodel_link_deletion>`：

- `restrict`：禁止删除目标对象。

所以如果 `MinorVampire` 的声明如下所示：

```sdl
type MinorVampire extending Person {
  former_self: Person {
    on target delete restrict;
  }
}
```

那么，只要 `NPC` 露西链接到了 `MinorVampire` 露西，`NPC` 露西则不允许被删除。

- `delete source`：在这种情况下，删除 `NPC` 露西（链接的目标）将自动删除 `MinorVampire` 露西（链接的来源）。

- `allow`：这个只是简单地允许你删除目标（这是默认设置），即 `NPC` 露西。

- `deferred restrict`：禁止删除目标对象，除非它在事务（transaction）结束时不再是目标对象。因此，这个选项类似于 `restrict`，但具有更大的灵活性。

所以如果你想让所有的 `MinorVampire` 对象在它们所属 `Vampire` 死亡时自动被删除，你可以给 `MinorVampire` 添加一个指向 `Vampire` 类型的链接。然后你可以使用 `on target delete delete source`：`Vampire` 是链接的目标，`MinorVampire` 是被删除的源。

接下来，让我们看一些查询的小技巧。

## 使用 distinct

`distinct` 很简单：只需将 `select` 更改为 `select distinct` 即可获得去重的结果。如果现在执行 `select Person.strength;`，我们可以看到结果中会有相当多的重复项，因为它是查询所有 `Person` 对象的力量值，且有些对象的力量值是相等的：

```
{5, 4, 4, 4, 4, 4, 10, 2, 2, 2, 2, 2, 2, 2, 3, 3}
```

如果将其更改为 `select distinct Person.strength;`，则输出为 `{2, 3, 4, 5, 10}`。

`distinct` 是按项目（item）工作，不会（对集合中的数组）做解包（unpack），因此 `select distinct {[7, 8], [7, 8], [9]};` 返回的是 `{[7, 8], [9]}` 而不是 `{7, 8, 9}`。

## 使用 `__type__`

我们之前已经了解到，我们可以使用 `__type__` 来获取查询中的对象类型，并且 `__type__` 里总是有 `.name` 来显示该类型的名称（否则我们只会得到 `uuid`）。就像我们可以通过 `select Person.name` 获得所有对象的名字一样，我们可以像下面这样来获得所有类型的名字：

```edgeql
select Person.__type__ {
  name
};
```

它向我们展示了至今为止扩展自 `Person` 的所有类型：

```
{
  schema::ObjectType {name: 'default::NPC'},
  schema::ObjectType {name: 'default::Crewman'},
  schema::ObjectType {name: 'default::MinorVampire'},
  schema::ObjectType {name: 'default::Vampire'},
  schema::ObjectType {name: 'default::PC'},
  schema::ObjectType {name: 'default::Sailor'},
}
```

或者我们也可以在常规查询中使用它来返回类型。让我们来看看有哪些类型下有名为 `Lucy` 的对象：

```edgeql
select Person {
  __type__: {
    name
  },
  name
} filter .name like 'Lucy%';
```

下面向我们展示了匹配到的对象，它们当然是 `NPC` 和 `MinorVampire`。

```
{
  default::NPC {__type__: schema::ObjectType {name: 'default::NPC'}, name: 'Lucy Westenra'},
  default::MinorVampire {__type__: schema::ObjectType {name: 'default::MinorVampire'}, name: 'Lucy'},
}
```

当结果里不包含类型信息时（例如，当结果为 JSON 格式时），使用 `__type__` 则非常有效。另外，这里有一个设置可以用来将输出格式切换为 JSON：即键入 `\set output-format json-pretty`。如果你如此操作后再重新执行前面的查询语句，你则会得到：

```
{"__type__": {"name": "default::NPC"}, "name": "Lucy Westenra"}
{"__type__": {"name": "default::MinorVampire"}, "name": "Lucy"}
```

恢复回默认格式类型可使用：`\set output-format default`。因为它非常方便，所以本书将继续使用默认格式选项显示结果。

## 内省

关键字 `introspect` 允许我们查看类型的更多详细信息。每种类型都有这些字段：`name`、`properties` 和 `links` 供我们访问，并且 `introspect` 可以让我们看到它们。现在让我们来试一下，看看会得到什么。我们将从 `Ship` 类型开始，它很小，但也包含了前面提到的三个字段。考虑到可能有人已经把它忘了，在此我们再一次列出 `Ship` 的属性和链接：

```sdl
type Ship {
  name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

首先，下面是最简单的 `introspect` 查询：

```edgeql
select (introspect Ship);
```

这个查询对我们来说不是很有用，但它确实说明了 `introspect` 是如何工作的：它返回了 `{schema::ObjectType {id: 28e74d09-0209-11ec-99f6-f587a1696697}}`。请注意，`introspect` 和类型要放在括号内；它是你要捕捉的类型的 `select` 表达式。

现在让我们将 `name`、`properties` 和 `links` 放入内省（introspection）中：

```edgeql
select (introspect Ship) {
  name,
  properties,
  links,
};
```

结果是：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 28e76c59-0209-11ec-adaa-85e1ecb99e47},
      schema::Property {id: 28e94e33-0209-11ec-9818-fb533a2c495f},
    },
    links: {
      schema::Link {id: 28e87ca8-0209-11ec-9ba8-71ef0b23db38},
      schema::Link {id: 28e8ee51-0209-11ec-b47e-8fd9b07debd3},
      schema::Link {id: 29176353-0209-11ec-a6c5-797987ef08b5},
    },
  },
}
```

就像直接在类型上使用 `select` 一样，如果输出包含另一种类型、属性等，我们只会得到一个 id。我们还是要指明我们想要什么。

因此，让我们在查询中添加更多内容以获取我们想要的信息：

```edgeql
select (introspect Ship) {
  name,
  properties: {
    name,
    target: {
      name
    }
  },
  links: {
    name,
    target: {
      name
    },
  },
};
```

于是，我们将得到：

1. `Ship` 的类型名称，
2. `Ship` 所含属性及其名称。同时我们也使用了 `target` 以获取属性指向的内容（即 `:` 之后的部分）。例如，`name: str` 的目标是 `std::str`。这里我们也想要了解目标类型的名称，因此我们同样对 `target` 使用了 `name`；如果没有 `name` 部分，我们将得到类似 `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}` 的输出（不具可读性）。
3. `Ship` 所含链接及其名称，以及链接的目标……以及目标的名称。

所有这些结合在一起，我们得到了一些可读和有用的东西。输出如下所示：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: '__type__', target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'crew', target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: 'sailors', target: schema::ObjectType {name: 'default::Sailor'}},
    },
  },
}
```

虽然这种查询 _看上去_ 很复杂，但编写的过程却十分简单——先尝试查询希望得到的内容，比如 properties，如果结果太诡异，就尝试加上 {name}，依次反复直到结果满意为止。

另外，如果查询本身不是太复杂（比如上面的例子），那么减少新行和缩进可能更有助于阅读。下面是同样的查询，但看起来简单多了：

```edgeql
select (introspect Ship) {
  name,
  properties: {name, target: {name}},
  links: {name, target: {name}},
};
```

[→ 点击这里查看到第 13 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何用一个单独的插入语句插入一个名为“Mr. Swales”，且到访过名为“York”的 `City`，名为“England”的 `Country` 以及名为“Whitby Abbey”的 `OtherPlace` 的 `NPC`？

2. 下面这个内省查询结果的可读性如何？

   ```edgeql
   select (introspect Ship) {
     name,
     properties,
     links
   };
   ```

3. 查看 `Vampire` 类型有哪些链接的最简单的方法是什么？

4. `select distinct {1, 2} + {1, 2};` 的输出会是什么？

   提示：别忘了笛卡尔乘法。

5. `select distinct {2, 2} + {2, 2};` 的输出会是什么？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _一位老朋友回来了。_
