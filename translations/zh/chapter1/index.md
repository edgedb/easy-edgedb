---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# 第一章 - 乔纳森前往特兰西瓦尼亚

在小说《德古拉》的开头，我们首先看到主人公乔纳森·哈克（Jonathan Harker），他是一位年轻的律师，正要去会见一个客户。这个客户是一位生活在东欧某个地方的有钱人——德古拉伯爵（Count Dracula）。乔纳森并不知道德古拉其实是一只吸血鬼，所以他目前还很享受前往欧洲的旅行。小说开始于乔纳森旅行时写下的日记。内容如下，其中粗体的部分适合存入数据库：

> 记于**5月3日，比斯特里察（Bistritz）**：**5月1日 8:35 P.M.** 离开**慕尼黑（Munich）**，次日清晨抵达**维也纳（Vienna）**；本应在 6:46 抵达，但火车晚点了一个小时。从我在火车上瞥到的来看，**布达佩斯（Buda-Pesth）**似乎是一个很棒的地方。

## 架构及对象类型

以上已经是很多信息了，它们帮助我们开始思考我们的数据库架构。EdgeDB 使用的语言称为 EdgeQL，它用于定义（define）、变换（mutate）和查询（query）数据。它的内部是 {ref}`SDL (schema definition language)<docs:ref_eql_sdl>`，这使迁移变得更加容易，我们将在本书中学习。到目前为止，我们的架构需要以下内容：

- 城市或位置类型。我们可以创建的这些类型称为对象类型 {ref}`object types <docs:ref_datamodel_object_types>`，由属性和链接组成。一个城市类型应该有什么样的属性呢？可能有名字、地理位置、以及有时会使用的其他名称或拼写。比如，Bistritz（比斯特里察）现在叫做 Bistrița（该城市在罗马尼亚），而 Buda-Pesth（布达佩斯）现在会被写为 Budapest。
- 人类类型。目前一个人类类型需要属性“姓名”以及一种方式去追踪他所访问过的地方。

创建一个类型，只需要使用关键字 `type` 并跟随对应的类型名及花括号 `{}`。类型 `NPC`（非玩家角色/书中人物）的创建如下所示：

```sdl
type NPC {
}
```

现在你已经完成了一个类型的创建，但里面还什么都没有。我们可以在花括号中为 `NPC` 类型添加属性。如果属性是必须的，我们要在属性名前使用 `required` property；如果属性是可选的，我们使用 `property`。

```sdl
type NPC {
  required name: str;
  places_visited: array<str>;
}
```

属性 `required name` 意味着：使用 `NPC` 这个类型创建的对象必须保证拥有一个“姓名（name）”，即你不能创建一个没有名称/姓名的 `NPC` 的对象，否则你会看到这样的错误提示：

```
MissingRequiredError: missing value for required property default::NPC.name
```

`str` 是指一个字符串，可以包含在单引号内：`'Jonathan Harker'` 或双引号内：`"Jonathan Harker"`。引号前使用转义字符 `\` 可使 EdgeDB 将引号视为一个字母打印出来，如：`'Jonathan Harker\'s journal'`。

`array` 是指相同类型的集合，即数组，我们这里的数组是一个 `str` 的数组。我们希望它看起来像这样：`["Bistritz", "Munich", "Buda-Pesth"]`。这样便于我们稍后搜索并查看哪个角色访问过哪里。

`places_visited` 不是一个 `required` 属性，因为我们之后可能会添加不会去任何地方的配角。比如“比斯特里察的某个客栈老板”或是其他什么人，我们并不了解或关心他的 `places_visited`。

现在让我们继续创建城市类型：

```sdl
type City {
  required name: str;
  modern_name: str;
}
```

这是类似的，只有带有字符串的属性。《德古拉》这本书出版于 1897 年，当时有些城市的名称拼写与现在有所不同。书中所有的城市都有名称（这就是为什么我们要在属性 `name` 前用 `required`），但有些城市并不需要不同的现代名称。比如，慕尼黑（Munich）至今仍然拼写为慕尼黑（Munich）。我们希望我们的游戏会将书中的城市名称与其现代名称关联起来，以便现在的我们可以轻松地在地图上找到它们。

## 迁移

目前我们尚未创建我们的数据库。在 [安装 EdgeDB](https://www.edgedb.com/download) 后，还有两个小步骤需要做。首先，我们需要创建一个项目 {ref}`"project" <docs:ref_quickstart_createdb>`，以便只有可以轻松地跟踪架构和处理迁移。然后，我们只需要通过运行 `edgedb` 来打开数据库控制台，从而将我连接到名为“edgedb”的默认数据库。接下来我们将使用它进行大量的实验。 

有时，你可能需要创建一个全新的数据库来尝试一些其他东西。此时，你可以通过键入关键字 `create database` 及你想要的数据库名称来创建它：

```edgeql
create database dracula;
```

然后我们键入 `\c dracula` 去连接该数据库。之后，你也可以通过键入 `\c edgedb` 再回到之前默认的数据库。

最后，我们需要做一个迁移。这将为数据库提供我们可以开始与之交互所需的结构。使用 EdgeDB 的内置工具 {ref}`built-in tools <docs:ref_cli_edgedb_migration>` 使得迁移并不困难。但是，这里我们将使用控制台快捷方式 {ref}`console shortcut <docs:ref_eql_ddl_migrations>` 来完成迁移：

- 首先，键入 `start migration to {}`。
- 在这个里面你至少要添加一个 `module`，你的类型才可以被访问。一个模块是一个命名空间，是相似类型聚集在一起的地方。`::` 左边的部分是模块的名称，右边是模块里包含的类型。如果你写了 `module default`，然后写 `type NPC`，`NPC` 类型将是 `default::NPC`。因此，例如，当你看到像 `std::bytes` 这样的类型时，它是指 `std`（标准库）中的类型 `bytes`。
- 然后添加我们上面提及的类型，并以 `}` 结尾来结束该表达块。然后在此之外，键入 `populate migration` 以添加数据。
- 最后，键入 `commit migration`，从而完成迁移。

按照上面所述，将它们放的一起，则如下所示：
```edgeql
start migration to {
  module default {
    type NPC {
      required name: str;
      places_visited: array<str>;
    }

    type City {
      required name: str;
      modern_name: str;
    }
  }
};

populate migration;
commit migration;
```

当然，除此之外，自然还有很多其他命令，尽管我们在本书中并不需要它们，但你可以将下面的四个页面添加至书签以供今后需要时使用：

- {ref}`Admin commands <docs:ref_cheatsheet_admin>`：创建用户角色，设置密码，配置端口等。
- {ref}`CLI commands <docs:ref_cheatsheet_cli>`：创建数据库，角色，为角色设置密码，连接数据库等。
- {ref}`REPL commands <docs:ref_cheatsheet_repl>`：主要介绍我们将在本书中使用的许多命令的快捷方式。
- {ref}`Various commands <docs:ref_eql_statements_rollback_tx>`：关于回滚事务、保存点声明等等。

我们还提供了以下编辑器的语法高亮插件，如果你愿意，可以通过相应的链接进行下载：[Atom](https://atom.io/packages/edgedb)，[Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb)，[Sublime Text](https://packagecontrol.io/packages/EdgeDB)，[Vim](https://github.com/edgedb/edgedb-vim).

下面则是我们刚刚创建的类型 `City`：

```edgeql
type City {
  required name: str;
  modern_name: str;
}
```

## 选择

在 EdgeDB 里，有三个包含了 `=` 的操作符，它们分别是：

- `:=` 用于声明；
- `=` 用于检查相等性 (注意：不是 `==`)；
- `!=` 是 `=` 相反的意思。

让我们用 `select` 来试试这些操作符。`select` 是 EdgeDB 里主要的查询命令，你可以使用它基于其后面的输入语句查询相应的结果。

顺便说一句，EdgeDB 里的关键字不区分大小写，因此使用 `select`，`select` 和 `SeLeCT` 都是一样的效果。但是使用大写字母是数据库的常规做法，因此我们将沿用这种常规做法。

首先，我们选择一个字符串：

```edgeql
select 'Jonathan Harker\'s journey begins.';
```

毫无意外，它将返回 `{'Jonathan Harker\'s journey begins.'}`，但你是否注意到它是在一个花括号 `{}` 里返回的？这个 `{}` 意味着它是一个集合，并且事实上，{ref}`everything in EdgeDB is a set <docs:ref_eql_everything_is_a_set>`（请确保记住这一点：“EdgeDB 里的一切都是一个集合”）。这也是为什么 EdgedB 里没有 null：在其他语言里会有 null，而在 EdgeDB 里你只会得到一个空集：`{}`。

接下来，我们来用 `:=` 为一个变量赋值：

```edgeql
select jonathans_name := 'Jonathan Harker';
```

它只是返回了我们给它的内容 `{'Jonathan Harker'}`。但是这次返回的是我们赋予的变量名为 `jonathans_name` 的字符串。

现在让我们用这个变量做一些事情。我们可以通过用关键字 `WITH` 来使用这个变量，然后用它和 `'Count Dracula'` 进行比较：

```edgeql
with jonathans_name := 'Jonathan Harker',
select jonathans_name != 'Count Dracula';
```

输出结果是 `{true}`。当然，你也可以写 `select 'Jonathan Harker' != 'Count Dracula'` 而得到同样的结果。接下来我们马上就会对我们用 `:=` 赋值的变量做一些事情。

## 插入对象

回到架构设计上，稍后我们会考虑给我们游戏中的城市增加时区和地理位置等信息。但现在，我们先来使用 `insert` 向数据库中插入一些数据。

别忘了要用逗号分割开各个属性，并用分号结束 `insert` 语句。此外，EdgeDB 同样倾向用两个空格作为缩进。

```edgeql
insert City {
  name := 'Munich',
};

insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};
```

注意：花括号里最后一项末尾的逗号是可选的——你可以加上逗号，也可以不加逗号。本书里，我们有时会在末尾添加一个逗号，有时也会将其省略。

最后，`NPC` 的插入如下所示：

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Munich", "Buda-Pesth"],
};
```

但是稍等，这个插入将不会链接到任何我们已经插入到数据库里的 `City`。这是我们的架构需要改进的地方：

- 我们有一个 `NPC` 类型和一个 `City` 类型，
- `NPC` 类型含有属性 `places_visited`，用来展示访问过的城市的名称，但它们只是数组中的字符串。我们最好能以某种方式将这个属性链接到 `City` 类型上。

所以这里我们先不急于做 `NPC` 的插入。我们先来调整一下 `NPC` 类型的定义，将 `array<str>` 从 `property` 更改为指向 `City` 类型的 `multi` 链接。这将使 `NPC` 和 `City` 连接起来。

但首先让我们先仔细看看当我们使用 `insert` 时到底发生了什么。

正如你所见，`str` 适用于像“ț”这样的 unicode 字母。甚至表情符号和特殊字符也都 OK：如果你愿意，你甚至可以创建一个名为“🤠”或“(╯°□°)╯︵ ┻━┻”的 `City`。

EdgeDB 还有一种字节字面量类型，可以用来创建用字符串表示的二进制数据。这主要是用于原始数据，通常这些原始数据并不是用来阅读的，存到文件里就好了。它们必须是 1 个字节长的字符。

你可以通过在字符串前添加一个 `b` 来创建一个字节文字：

```
db> select b'Bistritz';
{b'Bistritz'}
```

因为每个字符必须是 1 个字节，因此只有 ASCII 才适用于这种类型。所以如果 `modern_name` 是字节类型且名字中有类似 `ț` 这样的字符，将会产生错误：

```
db> select b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
  ┌─ query:1:8
  │
1 │ select b'Bistrița';
  │        ^ error
```

每当你 `insert` 一个项目（an item，一条数据）时，EdgeDB 都会给你一个 `uuid`。这是每个项目的唯一编号，类似于：

```
{default::NPC {id: 462b29ea-ff3d-11eb-aeb7-b3cf3ba28fb9}}
```

当你使用 `select` 选择一个类型时也会显示类似的内容。只需键入 `select` 和相应类型就会显示该类型下所有的 `uuid`。让我们看看到目前为止创建的所有城市：

```edgeql
select City;
```

这个命令返回了我们三个项目：

```
{
  default::City {id: 4ba1074e-ff3f-11eb-aeb7-cf15feb714ef},
  default::City {id: 4bab8188-ff3f-11eb-aeb7-f7b506bd047e},
  default::City {id: 4bacf860-ff3f-11eb-aeb7-97025b4d95af},
}
```

这仅说明我们已经有了三个 `City` 类型的对象。如果想查看对象内部，我们可以在查询语句里添加需要查看的属性或链接的名称。这称为“描述我们想要的数据的 {ref}`shape <docs:ref_eql_shapes>`。现在，我们选择所有 `City` 并通过下面这个查询显示他们的  `modern_name`：

```edgeql
select City {
  modern_name,
};
```

再一次说明，你不是必须在 `modern_name` 后面使用逗号，因为它是查询的末尾。

相信你还记得，书中提到的其中一个城市（慕尼黑 Munich）并没有与以前不同的现代名称 `modern_name`。但是它仍会显示一个“空集”，因为在 EdgeDB 中，所有数值都是元素的集合，即使里面没有任何东西。这里是上面查询的结果：

```
{
  default::City {modern_name: {}},
  default::City {modern_name: 'Budapest'},
  default::City {modern_name: 'Bistrița'},
}
```

因此有对象的 `modern_name` 是空集，同时另外两个对象的 `modern_name` 都有各自的名字。这再一次向我们展示了 EdgeDB 不存在其他一些语言里的 `null`，如果什么都没有，它会返回一个空的集合。 

现在第一个对象看起来是个谜（因为它不存在 `modern_name`），因此我们将在查询语句中添加 `name`，以便我们可以确认它就是城市“慕尼黑”（Munich）：

```edgeql
select City {
  name,
  modern_name
};
```

由此得到输出：

```
{
  default::City {name: 'Munich', modern_name: {}},
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

如果你只想返回类型的单个部分，而不是整个对象结构，你可以在类型名称后使用 `.` 。比如，`select City.modern_name` 将输出结果：

```
{'Budapest', 'Bistrița'}
```

这种表达式类型称为 _路径表达式_ 或 _路径（path)_，因为它是通往内部数值的直接路径。并且每个 `.` 都意味着前往再下一个路径，如果还有再深一层数值需要跟踪的话。

如果需要，你也可以在所需名称后使用 `:=`，将诸如 `modern_name` 之类的属性名称更改为成任何其他所需名称。这些名称将作为变量名在输出中显示出来。例如：

```edgeql
select City {
  name_in_dracula := .name,
  name_today := .modern_name,
};
```

输出结果为：

```
{
  default::City {name_in_dracula: 'Munich', name_today: {}},
  default::City {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest'},
  default::City {name_in_dracula: 'Bistritz', name_today: 'Bistrița'},
}
```

架构里的内容不会因此发生变化——它只是一个在查询中使用的快速变量名。

顺便说一下，`.name` 是 `City.name` 的缩写。你也可以每次都写 `City.name`（这被称为 _fully qualified name_），但不是必需的。

所以，如果我们可以对 `.name` 做一个快速索引的属性 `name_in_dracula`，我们还可以做其他类似的事情吗？我们确实可以，为了简单易懂，我们在这里举一个例子：

```edgeql
select City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'This is a city in the book Dracula'
};
```

输出结果是：

```
{
  default::City {
    name_in_dracula: 'Munich',
    name_today: {},
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Buda-Pesth',
    name_today: 'Budapest',
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
  default::City {
    name_in_dracula: 'Bistritz',
    name_today: 'Bistrița',
    oh_and_by_the_way: 'This is a city in the book Dracula',
  },
}
```

注意：`oh_and_by_the_way` 的类型是 `str`，即使我们不是必须说明它。EdgeDB 是强类型的：一切都需要一个类型，且它不会尝试将不同类型混在一起。因此，如果你写 `select 'Jonathan Harker' + 8;`，EdgeDB 会直接拒绝执行并给出错误提示 `operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
`.

另一方面，EdgeDB 可以使用“类型推断”来猜测类型，这就是 EdgeDB 在上面的例子里所做的：它知道我们正在创建一个 `str`。之后我们很快就会看到如何改变类型以及使用不同的类型。

## 链接

所以现在剩下要做的最后一件事是将 `NPC` 中名为 `places_visited` 的 `property` 更改为 `link`。现在，`places_visited` 为我们提供了我们想要的城市名称，但将 `NPC` 和 `City` 链接在一起才更有意义。毕竟，`City` 类型里面有 `.name`，链接它们比重写 `NPC` 中的所有内容更好些。我们将把 `NPC` 改成这样：

```sdl
type NPC {
  required name: str;
  multi places_visited: City;
}
```

我们在 `link` 前写了 `multi`，因为一个 `NPC` 应该可以链接不止一个 `City`。`multi` 的相反是 `single`，使用 `single` 意味着仅允许链接一个对象。因为 `single` 是默认的，所以如果你仅仅写 `link`，EdgeDB 将把它视为 `single`。

现在当我们插入乔纳森·哈克（Jonathan Harker）时，他将被链接到类型 `City`。别忘了属性 `places_visited` 不是 `required` 的，所以我们仍然可以在插入时仅使用他的名字进行创建：

```edgeql
insert NPC {
  name := 'Jonathan Harker',
};
```

这会生成一个 `NPC` 类型的对象，其 `places_visited` 属性被链接到 `City` 类型且没有任何内容。让我们看看里面是什么：

```edgeql
select NPC {
  name,
  places_visited
};
```

执行后会输出：`{default::NPC {name: 'Jonathan Harker', places_visited: {}}}`

但是我们想要把乔纳森（Jonathan）关联到他所去过的城市。我们在做 `insert` 的时候，可以将 `places_visited` 变为 `places_visited := City`：

```edgeql
insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

我们没有做任何过滤，所以 EdgeDB 会把所有 `City` 类型的对象都放进来。现在让我们看看乔纳森（Jonathan）都去了哪些地方。尝试一下下面的代码，你会发现这还不完全是我们需要的：

```edgeql
select NPC {
  name,
  places_visited
};
```

执行后的输出:

```
{
  default::NPC {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {id: 4ba1074e-ff3f-11eb-aeb7-cf15feb714ef},
      default::City {id: 4bab8188-ff3f-11eb-aeb7-f7b506bd047e},
      default::City {id: 4bacf860-ff3f-11eb-aeb7-97025b4d95af},
    },
  },
}
```

这和我们理想的结果已经很接近了！因为我们没有提到 `City` 里的任何属性，所以我们只能得到一堆 Object ID。现在我们仅仅需要让 EdgeDB 知道我们实际上是想看到 `City` 类型中的 `name` 属性。为了得到想要的结果，我们需要在 `places_visited` 后面添加一个冒号，并将 `name` 放到随后的花括号中：

```edgeql
select NPC {
  name,
  places_visited: {
    name
  }
};
```

成功了！现在我们可以从输出结果中得到我们想要的了：

```
{
  default::NPC {
    name: 'Jonathan Harker',
    places_visited: {
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
    },
  },
}
```

当然，乔纳森·哈克（Jonathan Harker）已经成功被插入到数据库中并关联了每一个造访过的城市。现在我们只有三个 `City` 对象，所以这还没有什么问题。但是稍后我们将有更多的城市，并且不能对其他所有角色都使用 `places_visited := City`（因为他们造访过的城市列表并不一样）。为此，我们将需要用到 ``，我们将在下一章中学习如何使用它。

注意：如果你在这里多次插入了“Jonathan Harker”，你将会得到多个名为“Jonathan Harker”的 `NPC` 对象。你可能觉得这不太合理，但我们暂时先允许这样做，我们将会在[第七章](../chapter7/index.md)中学习如何限制数据库中不出现多个使用相同姓名的 `NPC`。

[→ 点击这里查看到第 1 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 输入下面的代码将会被返回错误，请尝试通过添加一个字符，使其返回结果为 `{true}`。

   ```edgeql
   with my_name = 'Timothy',
   select my_name != 'Benjamin';
   ```

2. 请尝试插入一个名为 Constantinople 的 `City`，其现代的命名为 İstanbul。
3. 请尝试显示数据库中所有城市的名称。（提示：使用单行代码就可以做到，并不需要使用 `{}`）
4. 请尝试 select 出所有城市的 `name` 和 `modern_name`，并将 `.name` 显示为 `old_name`，将 `.modern_name` 显示为 `name_now`。
5. 键入 `SelecT City;` 会发生错误吗？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森·哈克抵达罗马尼亚_
