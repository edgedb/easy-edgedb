---
tags: Ddl, Sdl, Edgedb Community
---

# 第二十章 - 最后一战

终于，我们来到了最后一章——恭喜！下面是本章的最后一幕，但我们并不打算在本书中剧透最终的结局：

> 米娜（Mina）现在几乎已经是一只吸血鬼了，她说无论什么时候，她都能感觉到德古拉（Dracula）。范海辛（Van Helsing）带着米娜一起抵达了德古拉城堡，并让她在外面等候。范海辛进到城堡里，摧毁了吸血鬼女人和德古拉的棺材。与此同时，其他人正从南方赶来，也将抵达德古拉城堡。德古拉的“朋友们”把他放在棺材里，并用一辆马车正以最快的速度将他运回城堡。太阳快落山了，天空下起了雪，人们必须尽快抓到德古拉。他们不停追赶，越来越接近，终于抢到了棺材。他们拔出钉子打开了棺材，看到德古拉正躺在里面。乔纳森立即拔出他的刀。但就在这时，太阳下山了。德古拉微笑着睁开了眼睛，然后……

如果你对最终的结局感到好奇，可以查看 [这里](https://www.gutenberg.org/files/345/345-h/345-h.htm#chap19) 并搜索“the look of hate in them turned to triumph”。

然而，我们可以确信的是女吸血鬼们已经被摧毁了，因此我们可以通过给她们一个 `last_appearance` 对她们做最后的变更。范海辛在 11 月 5 日摧毁了她们，因此我们将插入该日期。但不要忘记过滤掉露西——她并不属于居住在城堡里的三个女 `MinorVampire`。

```edgeql
update MinorVampire filter .name != 'Lucy'
set {
  last_appearance := <cal::local_date>'1893-11-05'
};
```

根据最后一场战斗中发生的事情，我们之后可能不得不对德古拉或是一些英雄做同样的事情……

## 审查架构

点击 [这里](code.md) 可以查看到截至到现在我们搭建的架构和插入的所有数据。

现在我们已经学习了 20 章，你应该对我们搭建的架构以及如何使用它已经有了很好的理解。现在，让我们从上到下再过一遍，以确保我们完全理解了它，并一起考虑在实际游戏中哪些部分是好的，哪些部分还需要改进。

一个架构（a schema）的第一部分始终是启动迁移的命令：

- 在代码中我们使用了命令 `start migration to {};`，但 {ref}` ``edgedb migration`` <docs:ref_cli_edgedb_migration>` 工具在实际的项目中可以提供更好的控制。
- `module default {}`：在我们的架构里只使用了一个模块（命名空间），但如果你愿意，你可以制作更多。你可以使用 `describe type as sdl`（或 `as text`）查看该模块。

这里有一个 `Person` 的例子，如下所示，它向我们显示了它所在的模块：

`abstract type default::Person`

对于真正的游戏，我们的架构可能会很大，包含各种模块。我们可能会看到各个类型在不同的模块里，比如 `abstract type characters::Person` 和 `abstract type places::Place`。

我们架构中的第一个类型叫做 `HasNameAndCoffins`，它是抽象的，因为我们不需要任何这种类型的实际对象，它被像 `Place` 这样的非抽象类型所扩展。在我们游戏中的每一个地方：

1. 都有一个名称（排重并限制长度不超过 30 个字符）；
2. 都有 0 到多个棺材（这很重要，因为没有棺材的地方，吸血鬼较少出没，所以对人类来说更安全）。

```sdl
abstract type HasNameAndCoffins {
  required coffins: int16 {
    default := 0;
  }
  required name: str {
    delegated constraint exclusive;
    constraint max_len_value(30);
  }
}
```

我们本可以对 `coffins` 属性使用 {eql:type}` ``int32`` <docs:std::int32>`, {eql:type}` ``int64`` <docs:std::int64>` or {eql:type}` ``bigint`` <docs:std::bigint>`，但我们考虑到不会有那么多棺材，所以这里使用 {eql:type}` ``int16`` <docs:std::int16>` 就足够了。

接下来是 `abstract type Person`。这个叫做“人”的类型是目前为止最大的类型，它为我们的角色完成了大部分的工作。幸运的是，所有吸血鬼曾经都是人，并且也拥有诸如 `name` 和 `age` 之类的东西，因此他们也可以由 `Person` 扩展出来。

```sdl
abstract type Person {
  first: str;
  last: str;
  title: str;
  degrees: str;
  required name: str {
    delegated constraint exclusive;
  }
  age: int16;
  property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
  property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
  strength: int16;
  multi places_visited: Place;
  multi lovers: Person;
  first_appearance: cal::local_date;
  last_appearance: cal::local_date;
}
```

`exclusive` 可能是最常见的约束 {ref}`constraint <docs:ref_datamodel_constraints>` 了，我们用它来确保每个角色都有一个唯一的（无重复的）名称。这样就足够了，是因为我们已经从书里了解到没有角色重名的情况。但是，如果有可能出现多个“乔纳森·哈克（Jonathan Harker）”或其他角色的名称，我们可以使用内置的 `id` 属性。这个内置的 `id` 是自动生成的，并且是独占的 `exclusive`。

像 `conversational_name` 这样的属性是 {ref}`计算（computed）属性 <docs:ref_datamodel_computed>`。在我们的例子中，我们也添加了诸如 `first` 和 `last` 之类的属性。虽然删除 `name` 并只对每个角色使用 `first` 和 `last` 听起来不错，但因为书中有太多不重要的角色使用的是非正式的代称，比如：`Woman 2`、`The innkeeper`等，因此，我们没有删除 `name`，而是添加了 `conversational_name` 以满足特殊需求。当然，现实生活中，我们在标准的用户数据库中只会使用 `first` 和 `last` 以及像 `email` 这样带有 `constraint exclusive` 的字段来确保所有用户都是唯一的。

每个属性都有一个类型（如 `str`、`bigint` 等）。 计算（computed）属性也有，但我们不需要告诉 EdgeDB 需要什么类型，因为计算式表达本身就构成了类型。例如，`pen_name` 用到了 `str` 类型的 `.name`，并添加更多其他的字符串，这当然会产生一个 `str`。其中用于将它们连接在一起的 `++` 称为级联 {eql:op}`concatenation <docs:strplus>`。

其中有两个链接是 `multi` 链接，如果没有 `multi`，则一个 `link` 只能指向一个对象。如果你不写 `multi`，它将是一个 `single link`，也意味着你可能需要在创建链接时使用 `assert_single()`，否则会出现类似的错误提示：

```
error: possibly more than one element returned by an expression for a computed link 'former_self' declared as 'single'
```

你也可以将 `lover` 改为 `single` link，这取决于你的设定，但因为书中设定一个角色可能造访多个地点，因此 `places_visited` 必须是 `multi` 链接。

对于 `first_appearance` 和 `last_appearance`，我们使用的是 {eql:type}`docs:cal::local_date`，因为我们的游戏设定在了特定时期内且仅在欧洲的一部分地区活动。而对于现代的用户数据库，我们更喜欢用 {eql:type}`docs:std::datetime`，因为它是感知时区且总是符合 ISO8601 的。

所以对于用户遍布全球的数据库来说，`datetime` 通常是最好的选择。然后，你可以使用 {eql:func}`docs:std::to_datetime` 之类的函数将五个 `int64`、一个 `float64`（用于秒）和一个 `str`（用于 [the timezone](https://en.wikipedia.org/wiki/List_of_time_zone_abbreviations)）转换为一个总是作为 UTC 返回的 `datetime`：

```
db> select std::to_datetime(2020, 10, 12, 15, 35, 5.5, 'KST');
....... # October 12 2020, 3:35 pm and 5.5 seconds in Korea (KST = Korean Standard Time)
{<datetime>'2020-10-12T06:35:05.500Z'} # The return value is UTC, 6:35 (plus 5.5 seconds) in the morning
```

还有一个与 `HasNameAndCoffins` 类似的抽象类型是：

```sdl
abstract type HasNumber {
  required number: int16;
}
```

我们只将它用于 `Crewman` 类型，`Crewman` 扩展自两个抽象类型：

```sdl
type Crewman extending HasNumber, Person {
}
```

这个 `HasNumber` 类型用于五个起初没有名字的 `Crewman` 对象。后来，我们使用 `HasNumber` 中的 `number` 为他们创建了基于数字名称：

```edgeql
update Crewman
set {
  name := 'Crewman ' ++ <str>.number
};
```

即使 `HasNumber` 很少被用到，它也可能在之后变得很有用。比如，对于游戏后期的类型，它可能会被用于市民或随机的 NPC：“店主 2”、“马车司机 12”等。

我们的吸血鬼类型扩展自 `Person`，而 `MinorVampire` 有一个指向 `Person` 的可选（单一）链接。这是因为一些角色最初是人类，然后“重生”成为了吸血鬼。有了这个格式，我们可以使用 `Person` 中的 `first_appearance` 和 `last_appearance` 属性让各个角色出现在或离开游戏。如果有人变成了 `MinorVampire`，我们可以将两者链接起来。

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire;
}

type MinorVampire extending Person {
  former_self: Person;
}
```

有了这个格式，我们可以像下面这样查询所有变成 `MinorVampire` 的人。

```edgeql
select Person {
  name,
  vampire_name := .<former_self[is MinorVampire].name
} filter exists .vampire_name;
```

在我们的游戏中，只有 Lucy 满足条件：`{default::NPC {name: 'Lucy Westenra', vampire_name: {'Lucy'}}}`。但如果我们愿意，我们可以将游戏扩展到更早的历史时期，并将那三个吸血鬼女性与对应的 `NPC` 类型联系起来，将他们的 `former_self` 指向对应的 `NPC`。

此外，我们还定义了两个枚举类型，用于 `PC` 和 `Sailor`：

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
type Sailor extending Person {
  rank: Rank;
}

scalar type Class extending enum<Rogue, Mystic, Merchant>;
type PC extending Person {
  required class: Class;
}
```

`ShipVisit` 是我们架构中“最酷炫”的两个类型之一。我们把之前创建但从未使用过的 `Time` 类型中的大部分内容都给了它。其中，有一个叫做 `clock` 的属性，是一个字符串，并以下面的方式被使用：

- 将其转换为 {eql:type}`docs:cal::local_time` 并赋予属性 `clock_time`，
- 使用切片获取它的前两个字符来并赋予属性 `hour`。因为它是一个字符串，所以即使像 `1` 这样的单个数字也需要用两位数书写，即“01”，以适应“小时数”的获取方式，
- 由另一个名为 `vampires_are` 的计算（computed）属性决定此时的吸血鬼状态是 'asleep' 还是 'awake'，这取决于我们上一条中的 `hour` 属性，且需要将其先转换为 `<int16>`。

```sdl
type ShipVisit {
  required ship: Ship;
  required place: Place;
  required date: cal::local_date;
  clock: str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
        else SleepState.Awake;
}
```

`NPC` 类型里是我们第一次看到 {ref}` ``overloaded`` <docs:ref_eql_sdl_links_overloading>` 的地方，它使得我们可以以不同于默认的方式来使用属性、链接和函数等。在这里，我们希望将 `age` 限制为 120 岁，并以不同于 `Person` 的方式使用 `places_visited` 链接，将其默认值设置为 `London`。

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120);
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

我们的 `Place` 类型说明了你可以根据需要进行任意多次的扩展。它是一个 `abstract type`，它扩展自另一个 `abstract type`，然后再扩展为其他类型，例如 `City`。

```sdl
abstract type Place extending HasNameAndCoffins {
  modern_name: str;
  important_places: array<str>;
}
```

`important_places` 属性仅在下面的插入中使用过：

```edgeql
insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

`important_places` 现在只是一个数组。我们可以保持这样的设定，因为我们还没有为酒店和公园等非常小的地方创建类型的需求。但是如果我们打算为这些地方创建一个新类型，那么我们应该把 `important_places` 变成一个 `multi` 链接，而我们的 `OtherPlace` 类型甚至可能也不再是完全准确的类型了，如 {ref}`annotation <docs:ref_eql_sdl_annotations>` 所示：

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns count! Use the Castle type for that';
}
```

因此，在真实的游戏中，我们会创建一些其他较小的位置类型，并将 `City` 内的 `important_places` 的链接到这些较小的位置。我们也可以将 `important_places` 移动到 `Place`，这样类似 `Region` 的类型也可以通过它链接到一些较小的地点。

关于注释：我们使用 `abstract annotation` 来添加新注释类别：

```sdl
abstract annotation warning;
```

因为默认情况下，类型只能有名为 `title`、`description` 或 `deprecated` 的 {ref}`注释 <docs:ref_datamodel_annotations>`。我们在本书中使用注释只是为了好玩和教学，毕竟没有其他人同时在处理我们的数据库，因此我们不需要考虑协作所需要的交流。但是，如果我们是在为一个由多人参与制作的游戏创建真实的数据库，我们就会需要尽可能添加注释，以确保所有人都知道该如何使用每种类型。

我们创建 `Lord` 类型只是为了展示如何使用 `constraint expression on`，它允许我们创建自定义的约束：

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord')) {
    errmessage := "All lords need \'Lord\' in their name";
  }
}
```

（我们可能会在真正的游戏中删除它，或者它可能会成为扩展自 `PC` 类型的 `Lord`，以便玩家角色可以选择身份成为领主、小偷、或侦探等等。）

`Lord` 类型使用了函数 {eql:func}`docs:std::contains`，如果字符串、数组等包含我们正在搜索的项目，则该函数返回 `true`。这里还使用了 `__subject__`，它指的是类型本身：在这种情况下，`__subject__.name` 的意思是 `Person.name`。关于 `constraint expression on` 的更多说明与示例，可查阅 {eql:constraint}`此处 <docs:std::expression>`。

另一种创建 `Lord` 的可能方法如下所示，因为 `Person` 具有名为 `title` 的属性：

```sdl
type Lord extending Person {
  constraint expression on (__subject__.title = 'Lord') {
    errmessage := "All lords need \'Lord\' in their title";
  }
}
```

这将取决于我们创建 `Lord` 类型时，名称是使用 `.name` 中的单个字符串，还是通过一个计算（computed）属性使用 `.first`、`.last`、`.title` 来形成的全名。

接下来，扩展自 `Place` 类型的还有 `Country` 和 `Region`，我们刚在上一章看到过，所以就不在这里再次回顾它们了。但是 `Castle` 有些特别：

```sdl
type Castle extending Place {
  doors: array<int16>;
}
```

回到第 7 章，我们在查询中使用了它来查看乔纳森（Jonathan）是否可以打破城堡中的某扇门并逃离城堡。想法很简单：乔纳森会对每一扇门都做尝试，只要他比其中任意一个门更有力量，那么他就可以逃离城堡。

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  doors := (select Castle filter .name = 'Castle Dracula').doors,
select jonathan_strength > min(array_unpack(doors));
```

但是，后来我们学习了 `any()` 函数，所以让我们看看如何在这里利用上它。使用 `any()`，我们可以将查询更改为：

```edgeql
with
  jonathan_strength := (select Person filter .name = 'Jonathan Harker').strength,
  doors := (select Castle filter .name = 'Castle Dracula').doors,
select any(array_unpack(doors) < jonathan_strength); # Only this part is different
```

当然，我们也可以创建一个函数来做同样的事情，因为我们已经知道如何编写函数以及如何使用 `any()`。由于我们是按名称进行的过滤（Jonathan Harker 和 Castle Dracula），因此该函数也将只接收两个字符串作为参数并执行相同的查询。

不要忘记，我们需要用 {eql:func}`docs:std::array_unpack`，因为函数 {eql:func}`docs:std::any` 只接收集合：

```sdl
std::any(values: set of bool) -> bool
```

因此，`select any({5, 6, 7} = 7);` 可以正常工作，并返回 `{true}`。但 `select any([5, 6, 7] = 7);` 将报错：

```
error: operator '=' cannot be applied to operands of type 'array<std::int64>' and 'std::int64'
```

接下来的类型是 `BookExcerpt`，它需要大量的插入，对书中的每一个部分都进行摘抄并写入。因此，我们选择使用 {ref}` ``index on`` <docs:ref_eql_sdl_indexes>` 作用于属性 `.excerpt`，从而加快查找的速度。记住，仅在需要的地方使用 `index on`：因为在它提高查找速度的同时也会使数据库整体变大。

```sdl
type BookExcerpt {
  required date: cal::local_datetime;
  required author: Person;
  required excerpt: str;
  index on (.excerpt);
}
```

接下来是另一个有趣且“酷炫”的类型 `Event`。

```sdl
type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  multi excerpt: BookExcerpt;
  location: tuple<float64, float64>;
  east: bool;
  property url := get_url() ++ <str>.location.0 ++ '_N_' ++ <str>.location.1 ++ '_' ++ ('E' if .east else 'W');
}
```

这个类型可能是最接近真实游戏的实际可用的类型。使用 `start_time` 和 `end_time`、`place` 和 `people`（加上 `url`），我们可以正确地安排什么角色在什么位置以及是何时。`description` 属性是为该游戏数据库的使用者提供的，如 `'The Demeter arrives at Whitby, crashing on the beach'` 这样的描述，用于在需要时对事件的查找。

架构中的最后两种类型是 `Currency` 和 `Pound`，它们也是我们在前两章刚刚创建的，所以就不在这里再次回顾它们了。

## 导览 EdgeDB 文档

现在你已经读到了本书的结尾，你大概会开始阅读 EdgeDB 文档了。下面，我们将通过一些提示来结束本书，以便你在查看文档时感觉熟悉且易于阅读。

### 句法

本书包含了很多 EdgeDB 文档的链接，例如类型、函数等。如果你尝试创建类型、属性等并且遇到问题，最好从语法部分开始。这部分展示了需要遵循的顺序以及你拥有的所有选项。

举个简单的例子，这里是 {ref}`创建模块的语法 <docs:ref_eql_sdl_modules>`：

```sdl-synopsis
module ModuleName "{"
  [ schema-declarations ]
  ...
"}"
```

你可以看到一个模块只是一个模块名称加上 `{}` 以及里面的所有东西（架构声明）。这并不难。

那么，对象类型呢？ 它们看起来像 {ref}`这样 <docs:ref_eql_sdl_object_types>`：

```sdl-synopsis
[abstract] type TypeName [extending supertype [, ...] ]
[ "{"
    [ annotation-declarations ]
    [ property-declarations ]
    [ link-declarations ]
    [ constraint-declarations ]
    [ index-declarations ]
    ...
  "}" ]
```

这对你来说应该很熟悉了：你需要使用 `type TypeName` 来启动。你可以在左侧添加 `abstract`，也可以在右侧添加 `extending` 来拓展某个类型，然后其他所有内容都放在随后的 `{}` 当中。

同时，{ref}`属性 <docs:ref_eql_sdl_props>` 更为复杂些，它包括三种类型：具体的（concrete）、计算的（computed）和抽象的（abstract）。让我们来看一下我们最熟悉的“具体的（concrete）”属性声明：

```sdl-synopsis
[ overloaded ] [{required | optional}] [{single | multi}]
  [ property ] name : type
  [ "{"
      [ extending base [, ...] ; ]
      [ default := expression ; ]
      [ readonly := {true | false} ; ]
      [ annotation-declarations ]
      [ constraint-declarations ]
      ...
    "}" ]
```

你可以将语法视为有助于确保声明顺序正确的指南。

### 深入探寻 DDL

DDL 主要在处理迁移时会看到，因为它非常适合表达增量更改。到目前为止，我们只提到了函数的 DDL，因为在需要时添加 `create` 来创建函数非常容易。

SDL: `function says_hi() -> str using('hi');`

DDL: `create function says_hi() -> str using('hi')`

大小写也无关紧要。

但是对于类型，DDL 需要更多的输入，使用诸如 `create`、`set`、`alter` 等关键字。使用 {ref}` ``edgedb migration`` <docs:ref_cli_edgedb_migration>` 工具使得我们只需要使用 SDL 就可以处理架构。

## EdgeDB 词法结构

你可能想要查看或收藏 {ref}`此页面 <docs:ref_eql_lexical>` 以供你在项目期间可以随时参考。它包含了 EdgeDB 的整个词汇结构，也包括对于像《EdgeDB 易经》这样的教科书来说可能有些枯燥的项目，诸如运算符的优先顺序、所有保留关键字、可在标识符中使用哪些字符等等。

## 获得帮助

帮助总是一条信息就能搞定：获得帮助的最佳方式是在 GitHub 上的 [我们的讨论板](https://github.com/edgedb/edgedb/discussions) 上发起讨论。你还可以在 [此处](https://github.com/edgedb/edgedb/issues/new/choose) 对 EdgeDB 进行提问，也同样可以在该页面对本书提出问题。

## 道别之时

希望你在这个故事中找到了学习的乐趣，并且可以熟练地在自己的项目中运用 EdgeDB 进行开发。现在，让我们用《指环王》中的一首诗来结束本书，关于生命的无限可能性。

> The Road goes ever on and on
> 漫漫长路
> 
> Down from the door where it began.
> 起于家门
> 
> Now far ahead the Road has gone,
> 路其修远
> 
> And I must follow, if I can,
> 紧随不息
> 
> Pursuing it with eager feet,
> 步履匆匆
> 
> Until it joins some larger way
> 直奔通途
> 
> Where many paths and errands meet.
> 歧路迭起
> 
> And whither then?
> 去向何方？
> 
> I cannot say.
> 我心彷徨
> 
> 翻译自石中歌老师，邓嘉宛老师和杜蕴慈老师译本。

再见，或者不再见，不管结果如何！再次感谢你的阅读。

最后，感谢本书的中文翻译者：戴晨，daichen.daisy@gmail.com
