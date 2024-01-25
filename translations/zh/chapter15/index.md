---
tags: Expression On, Error Messages
---

# 第十五章 - 开始追捕吸血鬼

> 乔纳森（Jonathan）回来是件好事，但他仍然处于受惊的状态。他不确定自己与德古拉（Dracula）之间发生的事情是真是假，他觉得自己可能疯了。直到后来他遇到了范海辛（Van Helsing），范海辛告诉他这一切都是真的。乔纳森听到后，重新变得坚强和自信。于是他们开始寻找德古拉。就在此时，有人了解到德古拉买下了苏厄德医生（Dr. Seward）收容所附近的豪宅“卡法克斯（Carfax）”。这就是为什么伦菲尔德（Renfield）会受到如此强烈的影响……他们趁太阳升起时搜查了德古拉的豪宅，发现了他睡觉用的箱子。于是，他们将 Carfax 里所有的箱子都摧毁了，但他们清楚，在伦敦仍然还有很多。如果他们不摧毁掉所有，德古拉白天就可以躲在里面休息，到了晚上太阳落山后便跑出来威胁着伦敦。

## 更多抽象类型

这一章中我们了解到了一些关于吸血鬼的常识：白天时，吸血鬼需要睡在装有圣土的棺材里。这就是为什么德古拉乘坐德米特号（the Demeter）时带了 50 个棺材。这些棺材对我们的游戏机制很重要，所以我们应该为其创建一个类型。考虑到：

- 世界上的所有地方要么有棺材要么没有，
- 有棺材（Has coffins）的地方说明吸血鬼可以进入并威胁到当地的人们，
- 如果一个地方有棺材，我们应该知道有多少。

看起来这应该是抽象类型的一个好例子，因此我们这样定义：

```sdl
abstract type HasCoffins {
  required coffins: int16 {
    default := 0;
  }
}
```

因为大多数地方不会有专门供吸血鬼使用的棺材，所以棺材数量默认为 0。`coffins` 属性是一个 `int16`，如果数字为 1 或更大，意味着吸血鬼可以留在附近。在我们游戏的机制中，我们设定吸血鬼的活动范围大约是距离棺材 100 公里以内的区域。这是因为吸血鬼的作息表通常如下所示：

- 太阳下山后，吸血鬼精神焕发地在棺材中醒来，晚上 8 点离开棺材，开始寻找并伤害人类；
- 因为夜晚才刚刚开始，所以吸血鬼不用担心，可以远离棺材去寻找目标。他们可以乘坐每小时行驶 25 公里的马车；
- 大约凌晨一两点时，吸血鬼会开始感到紧张。因为大约 5 小时后太阳就会升起。他们不得考虑是否有充裕的时间返回棺材。

因此，晚上 8 点到凌晨 1 点之间是吸血鬼可以自由远离的时间，以 25 公里/小时的速度，他们可以在棺材周围获得大约 100 公里的活动半径。这个距离上，即使是最勇敢的吸血鬼也会在凌晨 2 点开始往家跑了。

对于更复杂的游戏，我们可以假设吸血鬼的活动在冬天更加猖獗（因为黑夜更加漫长，所以活动半径可以扩大到约 150 公里），但在这里我们就不对这些细节做更深入的研究了。

完成抽象类型 `HasCoffins` 的创建后，有很多类型需要 `extending`（扩展）自它。首先，我们可以让 `Place` 扩展自它，这样就可以把它的属性同时也赋给所有扩展自 `Place` 的位置类型了，比如 `City`、`OtherPlace` 等等：

```sdl
abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  };
  modern_name: str;
  important_places: array<str>;
}
```

另外，船也是大到足够放置棺材的（毕竟德米特号承载了 50 个），所以我们也将 `Ship` 改为扩展自 `HasCoffins`：

```sdl
type Ship extending HasCoffins {
  name: str;
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

如果我们需要，我们现在可以创建一个快速函数来判断吸血鬼是否可以进入某一个地方：

```sdl
function can_enter(person_name: str, place: HasCoffins) -> optional str
  using (
    with vampire := assert_single((select Person filter .name = person_name)),
    has_coffins := place.coffins > 0,
      select vampire.name ++ ' can enter.' if has_coffins else vampire.name ++ ' cannot enter.'
    );
```

你可能注意到了，这个函数中的 `person_name` 实际上只接受一个字符串用于选择过滤一个 `Person`。所以我们可能会得到类似“Jonathan Harker cannot enter（乔纳森·哈克不能进入）”的输出（但乔纳森是个人类）。尽管其中使用了 `assert_single()`，我们也可能愿意相信运用此函数的用户能够正确地使用它，但依旧无法避免错误的发生。所以如果我们不能信任使用者，这里有一些更健壮的方法：

- 重载函数以获得两个签名，每种类型的吸血鬼均有一个对应的函数签名：

```sdl
function can_enter(vampire: Vampire, place: HasCoffins) -> optional str
function can_enter(vampire: MinorVampire, place: HasCoffins) -> optional str
```

- 或是创建一个抽象类型，如 `abstract type AnyVampire`，再让 `Vampire` 和 `MinorVampire` 都扩展自 `AnyVampire`。最后给 `can_enter` 增加新的函数签名：`function can_enter(vampire: AnyVampire, place: HasCoffins) -> optional str`。

第一种方法可能是更简单的选择，因为它不需要我们进行任何显式迁移（explicit migration）。

注意：另一个你需要信任用户的地方是函数的输入不为空。因为如果用户的输入是空，则函数本身并不会被调用。例如：

```sdl
function try(place: City) -> str
  using (
    select 'Called!'
  );
```

然后，我们尝试调用它：`select try((select City filter .name = 'London'));` 输出为 `Called!`，符合我们的预期。这个函数需要一个 `City` 类型的参数，我们传入了 London，虽然函数体内并未用到，但函数还是执行并输出了 `Called!`。

但是，注意这里的输入不是可选的，这意味着如果你执行 `select try((select City filter .name = 'Beijing'));`，输出则为 `{}`，并没有执行 `select 'Called!'`，因为我们从未在数据库中插入一个叫做 `Beijing` 的城市。所以如果我们希望函数无论如何都被调用，该怎么做呢？我们可以将函数的参数声明为 `optional`：

```sdl
function try(place: optional City) -> str
  using (
    select 'Called!'
  );
```

修改后，`City` 类型的 `place` 仍然没有在函数体里被使用，但是因为我们添加了 `optional` 给这个输入参数，使他成为了可选的，因此无论这个输入是否在我们的数据库中有对应的 `City`，这个函数都会执行 `select 'Called!'`。也就是，无论你输入的是我们在数据库中可以找到的“London”，还是我们在数据库中没有插入的“Beijing”，函数都会被执行并输出 `Called!`。

正如 {ref}`文档 <docs:ref_sdl_function_typequal>` 中所说：“在这种情况下，当相应的参数为空时，函数仍然会被正常调用。”

这里还有一个典型的例子，就是在第 12 中提到的合并运算符 `??`。我们可以在 {eql:op}`它的签名 <docs:coalesce>` 中看到 `optional`：

`optional anytype ?? set of anytype -> set of anytype`

以上就是一些关于如何设置函数的想法，当然它取决于你认为人们会如何使用它们。

现在，让我们来给往伦敦（London）放置一些棺材。按照原著所述，我们的英雄们在当天晚上摧毁了卡法克斯（Carfax）里的 29 具棺材，也就是说在伦敦还有 21 具棺材不知所踪。

```edgeql
update City filter .name = 'London'
set {
  coffins := 21
};
```

现在，我们终于有机会调用我们刚刚创建的函数，并看看它是否有效了：

```edgeql
select can_enter('Count Dracula', (select City filter .name = 'London'));
```

得到：`{'Count Dracula can enter.'}`

当然，可能你还会有一些其他改进 `can_enter()` 的想法，比如：

- 将属性 `name` 从 `Place` 和 `Ship` 移到 `HasCoffins`。函数的第二个参数改为接收一个字符串（即地点名称），函数签名为 `function can_enter(person_name: str, place_name: str) -> optional str`，函数体中通过该地点的名称字符串 `select` 到对应的地点对象，然后再使用其属性 `coffins`，给出类似 `{'Count Dracula can enter London.'}` 的结果。
- 给函数加一个日期类型的输入，以便我们可以先检查这个吸血鬼是否已经死了。例如，如果我们输入一个露西死后的日期，它只会显示类似 `vampire.name ++ ' is already dead on ' ++ <str>vampire.last_appearance ++ ' and cannot enter ' ++ city.name`。

## 更多约束

接下来让我们再看一下其他的约束。我们已经见过了 `exclusive` 和 `max_value`，这里还有 {ref}`一些其他的约束 <docs:ref_std_constraints>` 我们也可以使用。

其中一个叫做 `max_len_value` 的方法可以确保字符串不会超过特定长度。这也许可以用于我们在许多章节之前创建的 `PC` 类型。目前，我们只用该类型做过一次测试，因为我们还没有任何真实玩家。我们一直是在根据小说（《德古拉》）在我们游戏的数据库中创建 `NPC`。`NPC` 不需要这个约束，因为他们的名字已经确定了，但是 `max_len_value()` 可以用于 `PC` 以确保玩家不会使用疯狂（太长）的名字。因此，我们可以将 `PC` 类型的定义更改为：

```sdl
type PC extending Person {
  required class: Class;
  overloaded required name: str {
    constraint max_len_value(30);
  }
}
```

现在，当我们尝试插入一个名字太长的 `PC` 时，则会被拒绝，并得到错误提示：`ERROR: ConstraintViolationError: name must be no longer than 30 characters`。

另一个方便的约束叫做 `one_of`，它有点像枚举。在我们的架构中，可以使用到它的一个地方是我们的 `Person` 类型中的 `title: str;`。你一定还记得我们添加这个属性时是为了便于我们在需要时可以将各种称呼（名字、姓氏、头衔、学位……）组合在一起生成所需的名称。`one_of` 这个约束则可以确保使用 `title` 的用户不会给自己编造头衔：

```sdl
title: str {
  constraint one_of('Mr.', 'Mrs.', 'Ms.', 'Lord')
}
```

对我们来说，添加一个 `one_of` 约束可能不太值得，因为整本书中可能有太多的头衔（伯爵、先生等等）且作为游戏的制作者，我们会在创建 `NPC` 的时候自行保证头衔的正确性，因此这里我们就不做真实的变更了。

另一个你可以想到的 `one_of` 的用武之地是“月份”，因为这本小说的时间只涉及了同一年中的五月到十月。假设我们有一个用于生成日期的类型，那么你可以在其定义中使用该约束：

```sdl
month: int64 {
  constraint one_of(5, 6, 7, 8, 9, 10)
}
```

但这取决于游戏将如何设定，这里只是示意大家可以如何来做。

下面让我们来学习一种新的约束，它可能是 EdgeDB 中最有趣的约束：

## 更灵活的约束

还一种特别灵活的约束，叫做 {eql:constraint}` ``expression on`` <docs:std::expression>`，它允许我们添加任何我们想要的表达式（约束）。在 `expression on` 之后（的圆括号中）添加一定为“真”的表达式即可。

假设我们稍后会出于某种原因需要一个 `Lord` 类型，并且所有 `Lord` 类型的对象名称中都必须包含“Lord”这个词。为此，我们可以给该类型添加一个约束并使用一个名为 {eql:func}`docs:std::contains` 的函数，该函数的签名如下所示：

```sdl
std::contains(haystack: str, needle: str) -> bool
```

功能是：如果 `haystack`（一个字符串）包含 `needle`（通常是一个较短的字符串），则返回 `{true}`。

于是，我们可以像下面这样用 `expression on` 和 `contains()` 来编写这个约束：

```sdl
type Lord extending Person {
  constraint expression on {
    contains(__subject__.name, 'Lord')
  }
}
```

这里的 `__subject__` 是指类型本身。

现在，当我们尝试插入一个名字中没有“Lord”的 `Lord` 对象时，将无法成功：

```edgeql
insert Lord {
  name := 'Billy'
  # Other stuff..
};
```

但是如果 `name` 是“Lord Billy”（或“Lord” + 任何东西），它就会正常工作。

在此期间，让我们再来练习一下 `select` 和 `insert` 的同时执行，以便我们立即看到 `insert` 的结果。我们把 `Billy` 改为 `Lord Billy`，且比利勋爵（Lord Billy）十分富有，到访过我们数据库中的所有地方。

```edgeql
select (
  insert Lord {
    name := 'Lord Billy',
    places_visited := (select Place),
  }
) {
  name,
  places_visited: {
    name
  }
};
```

其中 `.name` 包含了子字符串 `Lord`，因此它能正常工作：

```
{
  default::Lord {
    name: 'Lord Billy',
    places_visited: {
      default::Country {name: 'Hungary'},
      default::Country {name: 'Romania'},
      default::Country {name: 'France'},
      default::Country {name: 'Slovakia'},
      default::Castle {name: 'Castle Dracula'},
      default::City {name: 'Whitby'},
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
      default::City {name: 'Exeter'},
      default::City {name: 'London'},
    },
  },
}
```

## 自定义错误信息

由于 `expression on` 非常的灵活，因此你可以用任何你能想到的方式来使用它，但调用者可能并不了解这些我们自定义的约束中所包含的规则；且他们也无法从那些系统自动生成的错误信息里（如下所示）得到什么有用的帮助或者更多的错误细节：

`ERROR: ConstraintViolationError: invalid Lord`

所以使用者并没有办法从这里得知问题在于 `name` 里缺少了 `Lord`。幸运的是，约束本身允许你通过使用 `errmessage` 来设定个性化的错误提示信息，比如：`errmessage := "All lords need 'Lord' in their name."`

从而使得错误提示变得更加有效：

`ERROR: ConstraintViolationError: All lords need 'Lord' in their name.`

下面是 `Lord` 类型现在的样子：

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord')) {
    errmessage := "All lords need \'Lord\' in their name";
  }
}
```

## 双向关联

回到第 6 章，我们当时给 `Vampire` 添加了链接到 `MinorVampire` 类型的 `multi link slaves`，并从 `MinorVampire` 中删除了 `master` 链接。我们删除 `master` 的一个原因是为了降低复杂度，另一个原因是因为它们彼此依赖，使得 `delete` 变得不可行。但现在我们知道如何使用反向链接了，因此如果我们想，我们随时可以使用反向链接给 `MinorVampire` 重新添加一个 `master`。

（注意：我们不会真的修改 `MinorVampire` 类型，这里是为了说明如何重新添加 `master` 到 `MinorVampire`）

当前 `Vampire` 和 `MinorVampire` 类型的定义是：

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire;
}

type MinorVampire extending Person {
  former_self: Person;
}
```

添加 `master` 链接的一种方式是：增加一个名为 `master_name` 的字符串属性，然后我通过名字的匹配查询到对应的 `Vampire` 并将结果赋给 `master` 链接：

```sdl
type MinorVampire extending Person {
  former_self: Person;
  required master_name: str;
  link master := (
    with master_name := .master_name
    assert_single(select Vampire filter .name = master_name));
};
```

注意：因为 `master` 是一个 `single link`，所以我们需要使用 `assert_single()`（否则它将无法工作）。但看起来，我们需要写的有点多，且需要使用者确保自行正确输入所需的 `master_name`。因此，另一种更简单的添加 `master` 链接的方式则是使用反向链接：

```sdl
type MinorVampire extending Person {
  former_self: Person;
  single link master := assert_single(.<slaves[is Vampire]);
};
```

如果我们仍然需要访问 `master_name` 的快捷方式，我们可以在上面的花括号 `{}` 中追加 `property master_name := .master.name;`：

```sdl
type MinorVampire extending Person {
  former_self: Person;
  link master := assert_single(.<slaves[IS Vampire]);
  property master_name := .master.name;
};
```

现在让我们来验证一下上面的方法是否工作。首先，创建一个名为“Kain”的吸血鬼，并同时插入被他控制的两个 `MinorVampire` 奴隶，他们分别叫做“Billy”和“Bob”。

```edgeql
insert Vampire {
  name := 'Kain',
  slaves := {
    (insert MinorVampire {
      name := 'Billy',
    }),
    (insert MinorVampire {
      name := 'Bob',
    })
  }
};
```

如果新的 `MinorVampire` 类型正常工作，我们应该能够通过 `MinorVampire` 中的 `master` 链接直接看到“Kain”，而不必再使用反向链接。现在，让我们马上试一下：

```edgeql
select MinorVampire {
  name,
  master_name,
  master: {
    name
  }
} filter .name in {'Billy', 'Bob'};
```

结果是：

```
{
  default::MinorVampire {
    name: 'Billy',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
  default::MinorVampire {
    name: 'Bob',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
}
```

完美！所有的信息都在这里了。同时，你会发现，`MinorVampire` 和 `Vampire` 不再因为互相链接而无法删除了。

[→ 点击这里查看到第 15 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何创建一个名为“Horse”的类型，且其属性 `required name: str` 的值只能是“Horse”？

2. 如何让用户们能了解到 `name` 需要被赋值“Horse”？如何提示他们？

3. 如何确保 `NPC` 类型的 `name` 长度始终在 5 到 30 个字符之间？

   先用 `expression on` 试试。

4. 如何创建一个名为 `display_coffins` 的函数来显示出所有棺材数量大于 0 的 `HasCoffins` 类型的对象？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _伦菲尔德能帮上忙吗？_
