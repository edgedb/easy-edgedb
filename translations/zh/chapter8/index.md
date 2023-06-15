---
tags: Multiple Inheritance, Polymorphism
---

# 第八章 - 德古拉乘船前往英格兰

终于，我们离开了德古拉城堡。下面则是本章中发生的事情：

> 一艘船从保加利亚（Bulgaria）瓦尔纳市（Varna）出发，驶入黑海。船上有 **船长（captain）、大副（first mate）、二副（second mate）、厨师（cook）** 和 **五名船员（five crew）**。德古拉也在船上，但没有人知道。一到晚上德古拉都会从棺材里出来，于是每天晚上都会有一个人消失。船上的人都感到十分害怕，但也不知道发生了什么或者该做些什么。其中一个人说他看到了一个奇怪的身影在甲板上走来走去，但其他人都不相信他。在抵达英格兰（England）惠特比市（Whitby）前的最后一天，船上只剩下了船长一人——其他人都消失了。船长知道了真相。他将双手绑在方向盘上，这样即使德古拉找到了他，船也能继续直行。第二天，惠特比（Whitby）的人们看见了一艘搁浅的船，一只狼从上面跳下来并跑到了岸上——那是德古拉的狼化身，但人们并不知道。人们发现了死去的船长，他依旧绑在方向盘上，手里还拿着一个笔记本，于是人们开始阅读里面的故事。

> 此时，米娜（Mina）和她的朋友露西（Lucy）恰好也在惠特比（Whitby）度假……

## 多重继承

在德古拉到达惠特比（Whitby）时，让我们来学习一下多重继承（multiple inheritance）。我们已经学习了如何在一个类型上 `extend` 出另一个类型，我们也已经对其进行了很多次的实践，比如：从 `Person` 扩展出 `NPC`，从 `Place` 扩展出 `City`，等等。多重继承是指同时对多个类型执行此操作。现在，让我们在船员身上做一下尝试。原著里他们并没有名字，所以我们用编号来表示他们。大多数 `Person` 类型不需要数字编号，因此我们将创建一个名为 `HasNumber` 的抽象类型，为那些需要数字编号的人物所使用：

```sdl
abstract type HasNumber {
  required number: int16;
}
```

此时，我们还需要从 `Person` 类型的 `name` 中删除 `required`。因为现在不是每个 `Person` 类型的对象都会有一个名字了，对于有名字的 `Person`，我们相信使用者自然会正确输入其对应的名字。当然，我们还是会继续保留 `exclusive`。

现在，让我们来运用多重继承为船员创建 `Crewman` 类型。这很简单：只需在要扩展的每个类型之间添加一个逗号：

```sdl
type Crewman extending HasNumber, Person {
}
```

现在我们有了 `Crewman` 并且不需要为他们赋予名字，只需要给他们一个编号，而函数 `count()` 可以使我们对船员的插入变得很容易。我们只需要重复执行五次下面的语句：

```edgeql
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
```

即，如果还没有任何 `Crewman` 类型的对象，新插入的第一个船员将获得编号 1，下一个将获得编号 2，依此类推。所以操作五次后，我们可以通过 `select Crewman {number};` 来查看结果了。我们将得到：

```
{
  default::Crewman {number: 1},
  default::Crewman {number: 2},
  default::Crewman {number: 3},
  default::Crewman {number: 4},
  default::Crewman {number: 5},
}
```

接下来为水手创建 `Sailor` 类型。水手是有等级的，所以我们先为此创建一个枚举：

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
```

现在，我们将使用 `Person` 和 `Rank` 来创建 `Sailor` 类型：

```sdl
type Sailor extending Person {
  rank: Rank;
}
```

然后，我们将制作一个 `Ship` 类型来承载他们（即船员和水手们）：

```sdl
type Ship {
  required name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

接下来，让我们来插入水手的信息，我们只需给他们一个名字并从枚举中选择一个等级：

```edgeql
insert Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

insert Sailor {
  name := 'Petrofsky',
  rank := 'FirstMate'
};

insert Sailor {
  name := 'The Second Mate',
  rank := 'SecondMate'
};

insert Sailor {
  name := 'The Cook',
  rank := 'Cook'
};
```

插入 `Ship` 很容易，因为现在所有 `Sailor` 和 `Crewman` 都是这艘船的一部分——我们不需要使用任何 `filter`：

```edgeql
insert Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

然后，我们可以通过 `select Ship` 来证实是否全部船员都已经“上船”：

```edgeql
select Ship {
  name,
  sailors: {
    name,
    rank,
  },
  crew: {
    number
  },
};
```

输出结果是：

```
{
  default::Ship {
    name: 'The Demeter',
    sailors: {
      default::Sailor {name: 'The Captain', rank: Captain},
      default::Sailor {name: 'Petrofsky', rank: FirstMate},
      default::Sailor {name: 'The First Mate', rank: SecondMate},
      default::Sailor {name: 'The Cook', rank: Cook},
    },
    crew: {
      default::Crewman {number: 1},
      default::Crewman {number: 2},
      default::Crewman {number: 3},
      default::Crewman {number: 4},
      default::Crewman {number: 5},
    },
  },
}
```

## 序列类型

EdgeDB 有一个叫做 {eql:type}`docs:std::sequence` 的类型，它对于给一个类型的对象进行编号十分有用。这个类型被定义为“int64 的自动递增序列”，因此一个 `int64` 从 1 开始，每次被使用都会加 1。假设我们有一个 `Townsperson` 类型，并尝试对其使用“sequence”。但我们不能像下面这样简单地编写：

```sdl
type Townsperson extending Person {
  number: sequence;
}
```

这样是行不通的，因为 `sequence` 会始终记录最新的递增结果，如果每种类型都直接使用 `sequence`，那么它们就会共享它（即每个类型被使用的时候都会对其加 1）。所以正确的做法是将其扩展为你命名的另一种类型，然后该类型将从 1 开始计数。所以我们的 `Townsperson` 类型应该按如下编写：

```sdl
scalar type TownspersonNumber extending sequence;

type Townsperson extending Person {
  number: TownspersonNumber;
}
```

删除已经插入的项目（对象），对 `sequence` 类型的数字继续加 1 并不产生影响。比如，如果你插入五个 `Townsperson` 对象，它们的数字 `number` 将是从 1 到 5。然后，如果此时将它们全部删除，然后又插入了一个 `Townsperson`，那么这个数字将会继续增加至 6（而不是重新回到 1）。所以对于 `Crewman` 类型的创建，使用 `sequence` 也是一个很好的选择，它非常方便，并且没有重复的可能，每次插入时数字都会自行加 1。当然，如果你非要这么做，你也 _可以_ 通过使用 `update` 和 `set` 生成重复的数字（EdgeDB 不会阻止你），但即便如此，当你进行下一次插入时它仍然会跟踪使用仍在计数的下一个数字。

## 使用 is 查询多类型

现在我们已经有相当多的类型扩展自 `Person` 类型，其中很多都有自己的属性。`Crewman` 类型有属性 `number`，同时 `NPC` 类型有一个叫做 `age` 的属性。

如果我们想同时查询它们，该怎么办呢？他们都扩展自 `Person`，但 `Person` 本身并没有这些子类型的所有链接和属性。因此，下面这个查询并不会起作用：

```edgeql
select Person {
  name,
  age,
  number,
};
```

错误提示是：`ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`。

幸运的是，EdgeDB 提供了一个简单的解决方法：我们可以在方括号内使用 `IS` 来指定类型。具体做法是：

- `.name`：可以保持不变，因为 `Person` 有这个属性；
- `.age`：属于 `NPC` 类型，所以将其更改为 `[is NPC].age`；
- `.number`：属于 `Crewman` 类型，因此将其更改为 `[is Crewman].number`。

这样便可以正常工作了：

```edgeql
select Person {
  name,
  [is NPC].age,
  [is Crewman].number,
};
```

输出的内容很多，所以这里只展示其中的一部分。你会注意到没有对应属性或链接的类型会返回一个空集：`{}`：

```
{
  # ... /snip
  default::Crewman {name: {}, age: {}, number: 4},
  default::Crewman {name: {}, age: {}, number: 5},
  default::PC {name: 'Emil Sinclair', age: {}, number: {}},
  default::NPC {name: 'The innkeeper', age: 30, number: {}},
  default::NPC {name: 'Mina Murray', age: {}, number: {}},
  default::NPC {name: 'Jonathan Harker', age: {}, number: {}},
}
```

现在看起来还不错，但是如果我们将此输出作为 JSON 发送到某个地方，它不会向我们显示这些对象分别是什么类型。那么，要在 EdgeDB 的查询中引用对象本身的类型信息，你可以使用 `__type__`。只调用 `__type__` 则只能得到一个 `uuid`，所以我们需要添加 `{name}` 来表明我们想要的是类型的名称。如果你想在查询中显示对象的类型，则可以访问这个字段，因为所有类型都有 `name` 这个字段。

```edgeql
select <json>Person {
  __type__: {
    name # Name of the type inside module default
  },
  name, # Person.name
  [is NPC].age,
  [is Crewman].number,
};
```

这里我们仅展示输出结果的前六个对象：

```
{
  # ... /snip
  {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": null, \"number\": 4, \"__type__\": {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": null, \"number\": 5, \"__type__\": {\"name\": \"default::Crewman\"}}",
  "{\"age\": null, \"name\": \"Emil Sinclair\", \"number\": null, \"__type__\": {\"name\": \"default::PC\"}}",
  "{\"age\": 30, \"name\": \"The innkeeper\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
  "{\"age\": null, \"name\": \"Mina Murray\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
  "{\"age\": null, \"name\": \"Jonathan Harker\", \"number\": null, \"__type__\": {\"name\": \"default::NPC\"}}",
}
```

上面的操作被称为多态查询 {ref}`polymorphic query <docs:ref_eql_select_polymorphic>`，这也是我们在架构中使用抽象类型的理由之一.

## 超类型、子类型及泛型类型

我们将那些被其他类型扩展的类型称为 `supertype`（超类型）。扩展出的类型是它们的 `subtypes`（子类型）。因为继承一个类型会得到它的所有特性，所以`subtype is supertype` 将返回 `{true}`。反之，`supertype is subtype` 将返回 `{false}`，因为超类型不继承其子类型的特性。

在我们的架构中，这意味着 `select PC is Person` 返回 `{true}`，而 `select Person is PC` 将返回 `{true}` 或 `{false}`，具体取决于所选对象是否是 `PC`。

想要对这一信息进行确认，只需增加一个计算（computed）属性来查询 `Person is PC`，具体操作如下所示：

```edgeql
select Person {
    name,
    is_PC := Person is PC,
};
```

那么对于更简单的标量类型会怎样？我们知道 EdgeDB 在整数、浮点数等不同类型方面是非常精确的，但是如果你只是想知道一个数字是否是整数呢？我们同样可以使用 `IS`，但看起来有点冗余：

```edgeql
with year := 1893,
select year is int16 or year is int32 or year is int64;
```

输出结果是：`{true}`.

幸运的是，`int16` 这些类型都是从 {ref}`抽象类型 <docs:ref_std_abstract_types>` 扩展出来的，我们可以使用这些抽象类型，它们都以 `any` 开头，包括 `anytype`，`anyscalar`，`anyenum`，`anytuple`，`anyint`，`anyfloat`，`anyreal`。唯一可能让你不确定的是 `anyreal`：它意味着任何实数，包括整型和浮点型，以及 `decimal` 类型。

因此，你可以将上述输入简化为 `select 1893 is anyint` 并获得 `{true}`。

## Multi 的使用

我们已经看到过很多次 `multi` 链接了，你可能想知道 `multi` 是否也可以用在其他地方。答案是肯定的。比如 `multi`的 property，它与任何属性并无二致，但它可以有多个值。例如，我们的 `Castle` 类型有一个用于 `doors` 属性的 `array<int16>`：

```sdl
type Castle extending Place {
  doors: array<int16>;
}
```

为了达到同样效果，我们也可以将其写为：

```sdl
type Castle extending Place {
  multi doors: int16;
}
```

这样一来，当你进行插入，对 `doors` 进行赋值时，你需要使用的是 `{}`，而不是使用方括号的数组：

```edgeql
insert Castle {
  name := 'Castle Dracula',
  doors := {6, 19, 10},
};
```

接下来你可能会问，使用哪种方法更好？`multi` property 还是 `array`？或是使用 `multi` 链接到对象。答案是……视情况而定。但是这里有一些很好的经验法则可以帮助你做决定。

- `multi` property 与数组：

  取决于你正在处理的数据有多大？当你的数据量很大时，`multi` property 更有效，而数组则较慢。但是如果你处理的集合较小，那么数组比 `multi` property 更快。

  如果你想对单个元素使用索引（indexes）和约束（constraints），那么你应该使用 `multi` property。我们将在第 16 章中学习如何在查询种使用索引，现在只需要知道它们是一种加快查询的方法。

  如果顺序很重要，那么使用数组可能会更好。因为保持数组中项目的原始顺序更加容易。

- `multi` property 与对象

  我们先来看使用 `multi` property 更好的两个方面，即使用对象的劣势，然后再说使用对象的好处。

  首先，对象的第一个负面问题是：对象总是很大。还记得 `describe type as text` 吗？让我们用它来看看我们之前创建的 `Castle` 类型：

  ```
  {
    'type default::Castle extending default::Place {
      required single link __type__: schema::Type {
          readonly := true;
      };
      optional single property doors: array<std::int16>;
      required single property id: std::uuid {
          readonly := true;
      };
      optional single property important_places: array<std::str>;
      optional single property modern_name: std::str;
      required single property name: std::str;
  };',
  }
  ```

  你一定看到过 `readonly := true` 的类型，你创建的每个对象类型都会有它们。`__type__` 链接和 `id` 属性分别都是 16 字节。

  对象的第二个负面问题是类似的：对象更多是为计算机工作的。EdgeDB 运行在 PostgreSQL 之上，指向对象的 `multi` 链接需要额外的“连接（join）”（链接表 + 对象表），但 `multi` property 并不需要。此外，“反向链接“（backlink）（你将在第 14 章中学习到）也需要更多类似的额外工作。

  现在，我们来介绍一下经过对比后，使用对象的两个好处：

  是否有其他类型需要引用相同的值？如果是这样，那么最好使用对象来保持一致。这就是为什么我们最终将 `places_visited` 设为 `multi` 链接的原因。

  如果你需要为每个对象设置多个值，则使用对象更容易进行迁移/变更（migrate）。

希望这些解释对你有所帮助。大多数时候，记住以上几点经验应该可以帮助你在众多选择中了解到自己真正的需要并做出更明智的决定。

[→ 点击这里查看到第 8 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何选择出所有 `Place` 和他们的名字，如果当它是个 `Castle` 时，同时显示属性 `doors`？

2. 如何在选择 `Place` 时，用 `city_name` 显示当它是个 `City` 时的 `name`，并用 `country_name` 显示当它是个 `Country` 时的 `country_name`？

3. 基于上一题，如果只想显示属于 `City` 或 `Country` 类型的结果，该如何做？

4. 如何显示所有没有 `lover` 的 `Person` 对象及其姓名和所属类型的名称？

5. 下面这个查询需要修复什么？提示：有两个地方是必须要修复的，还有一个地方可以修改得更具有可读性。

   ```edgeql
   select Place {
     __type__,
     name
     [is Castle]doors
   };
   ```

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候认识苏厄德医生、亚瑟霍姆伍德和昆西莫里斯了……还有奇怪的伦菲尔德。_
