---
tags: Scalar Types, Abstract Types, Filter
leadImage: illustration_02.jpg
---

# 第二章 - 比斯特里茨的酒店

在这一章中，我们将继续阅读这个故事，并思考需要将哪些信息存入数据库。下面叙述中的粗体文字均为重要的信息：

> 乔纳森·哈克（Jonathan Harker）在 **比斯特里茨（Bistritz）** 发现了一家酒店, 叫做 **金克朗酒店（Golden Krone Hotel）**。他在酒店里收到了一封来自德古拉（Dracula）的欢迎信，信中提到德古拉正在 **城堡（castle）** 里等他。乔纳森·哈克第二天不得不搭乘马车才能到达那里。同时我们还了解到乔纳森·哈克来自 **伦敦（London）**。金克朗酒店（Golden Krone Hotel）的老板似乎很害怕德古拉。他不想让乔纳森（Jonathan）离开并表示前往城堡会很危险，但乔纳森并没有听进去。一位老太太给了乔纳森一个金色的十字架，说十字架可以保护他。乔纳森感到很尴尬，认为这可能是出于礼貌，他并不知道之后这会对他有多大的帮助。

现在我们开始看一下比斯特里茨（Bistritz）这座城市的细节。通过阅读上面的情节，你可能会想到我们可以给 `City` 添加一个叫做 `important_places` 的属性。它可以是像 **金克朗酒店（Golden Krone Hotel）** 这样的地方。虽然我们尚不确定这些地方在将来是否会拥有属于自己的类型，但至少现在不需要，所以我们暂时只是将它定义为一个字符串的数组：`important_places: array<str>;` 然后我们则可以把这些重要地点的名字放进去。也许之后还会发展出更多的内容，但目前为止，`City` 的定义暂时如下所示：

```sdl
type City {
  required name: str;
  modern_name: str;
  important_places: array<str>;
}
```

现在，我们来为比斯特里茨（Bistritz）插入一条数据：

```edgeql
insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

## 枚举、标量类型及类型扩展

在我们的游戏中，必须有玩家角色类。这本书以 1893 年为背景，所以我们的玩家角色将拥有适合 19 世纪后期的色类。这里用 `enum`（枚举）可能是最好的选择，因为 `enum` 可以提供多个选项，供使用者选择一个所需的。枚举的变量需要用大写驼峰式（UpperCamelCase）进行书写。

这里，我们将第一次看到关键词 `scalar`：它是一个“标量类型”（`scalar type`），因为一次只保存一个值。而其他类型（如 `City`、`NPC`）属于“对象类型”（`object types`），因为他们能够同时保存多个值。

另一个我们将第一次看到的关键词是 `extending`：它是指以一个类型作为基础并对其进行扩展。这不仅为你提供了你想要扩展的类型的所有功能，并允许你添加更多选项。因此，我们将这样定义 `Class` 类型：

```sdl
scalar type Class extending enum<Rogue, Mystic, Merchant>;
```

(Rogue, Mystic, Merchant就是盗贼、神秘主义者、商人呢)

你是否留意到 `scalar type` 的定义是以一个分号结尾的，而其他类型并非如此？这是因为其他类型用 `{}` 构成了一个完整的表达式。但是这里的单行代码我们并没有 `{}`，所以在这里我们需要用分号来说明表达式的结束。

要在枚举中的变体（选项）间进行选择，只需用 `.`。对于上面的枚举，这意味着我们可以选择 `Class.Rogue`、`Class.Mystic` 或 `Class.Merchant`。

现在设定这个 `Class` 类型将被我们的游戏玩家所扮演的角色使用，而不是被书中已有的故事人物所使用（因为他们的故事和选择已成定局）。这意味着我们需要一个 `PC` 类型和一个 `NPC` 类型。因为`PC`和`NPC`彼此非常相似，我们可以创造一个 `abstract type Person`（抽象类型）。我们的 `Person` 类型应该保留——我们可以将它用作两者的基本类型。为此，我们可以让 `Person` 成为一个 `abstract type`（抽象类型）而不仅仅是一个 `type`。有了这个抽象类型，我们可以对 `PC` 和 `NPC` 类型的定义使用关键字 `extending`。

因此，现在这部分结构看起来像这样：

```sdl
abstract type Person {
  required name: str;
  multi places_visited: City;
}

type PC extending Person {
  required class: Class;
}

type NPC extending Person {
}
```

现在书中的角色都将是 `NPC`类型，而 `PC` 是在考虑到这是个游戏的情况下设定的。`Person` 是一个抽象类型，因此我们不能再对其进行直接的插入。如果你尝试执行 `insert Person {name := 'Mr. HasAName'};`，你将会收到错误提示：

```
error: QueryError: cannot insert into abstract object type 'default::Person'
  ┌─ <query>:1:8
  │
1 │ insert Person {name := 'Mr. HasAName'};
  │        ^^^^^^ error
```

但只要你将 `Person` 改为 `NPC`，它就可以工作了。

此外，`select` 一个抽象类型是没有问题的，它会选择出所有从该抽象类型扩展出来的类型。

现在让我们也来操作一下玩家角色。我们创建一个名叫 Emil Sinclair 的人，他是一个神秘主义者。我们也将 `City` 直接赋值给他的 `places_visited`，于是他也拥有了那三个乔纳森造访过的城市。

```edgeql
insert PC {
  name := 'Emil Sinclair',
  class := Class.Mystic,
};
```

`places_visited := City` 是对 `places_visited := (select City)` 的简写，你不是必须每次都输入 `select` 部分。

请注意，我们并没有只写 `Mystic`，我们必须写明枚举类型 `Class` 并选择其中一个枚举值。

## 类型转换

Casting 是指快速将一种类型转换为另一种类型，它在 EdgeDB 中被大量使用，因为 EdgeDB 对类型很严格，并拒绝对两种不同的类型进行操作。但为了方便，很多类型转换都是自动完成的。例如：

```edgeql
select 9 + 9.9;
```

EdgeDB 不会在此处生成错误，只会返回一个 `float64` 类型的正确输出 `18.9`。你可以通过下面语句进一步印证：

```edgeql
select (9 + 9.9) is float64;
```

执行后返回 `true`；如果执行 `select (9 + 9.9) is float32;` 将返回 `false`。

当你需要自己进行类型转换时，你可以使用 `<>` 尖括号指明“转向”的类型。例如，执行下面的语句将产生一个错误：

```edgeql
select '9' + 9;
```

EdgeDB 在这里会告诉我们确切的问题是：

```
error: operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'
  ┌─ query:1:8
  │
1 │ select '9' + 9;
  │        ^^^^^^^ Consider using an explicit type cast or a conversion function.

```

要修复它，只需要使用 `<>` 尖括号指明要将字符串 `'9'` 转换为 `int32` 类型：

```edgeql
select <int32>'9' + 9;
```

然后你会得到 `18`，一个 32 位整数。

如果需要，你可以一次性转换多次。下面这个例子并非常规做法，这里只是为了展示：如果你愿意，你可以如何一遍又一遍地进行类型转换：

```edgeql
select <str><int64><str><int32>50 is str;
```

执行后会返回 `{true}`，因为我们所做的只是询问它是否是一个 `str`，且它确实是。

类型转换从右往左执行，最后的转换是在最左侧。因此，`<str><int64><str><int32>50` 意味着：50 先变成了 int32，再变成了 str，又变成了 int64，最后又变成了 str。

此外，需要注意：类型转换仅适用于标量类型 `scalar type`；而用户创建的对象类型，如 `City` 和 `Person` 等都过于复杂，并不能简单地进行相互转换。

## 过滤器

最后，在我们结束第 2 章前，让我们来一起学习下如何使用 `filter`。你可以在 `select` 的花括号后面使用 `filter` 来控制只显示某些结果。现在，让我们试着用 `filter` 来过滤并显示名为”Emil Sinclair“的 `Person` 对象：

```edgeql
select Person {
  name,
  places_visited: {name},
} filter .name = 'Emil Sinclair';
```

`filter .name` 是 `filter Person.name` 的缩写。如果你愿意，你也可以写成 `filter Person.name`，它们是一样的。

输出结果如下：

```
{
  default::PC {
    name: 'Emil Sinclair',
    places_visited: {},
  },
}
```

现在让我们来试着过滤城市。这里有一种灵活的搜索方式，是使用 `like` 或 `ilike` 来匹配字符串的一部分。

- `like` 是区分大小写的：“Bistritz”可以匹配“Bistritz”，但和“bistritz”并不匹配。
- `ilike` 是不区分大小写的（ilike 中的 I 是指**不敏感（insensitive）**），所以“Bistritz”可以匹配“BiStRitz”，也可以匹配“bisTRITz”。

你也可以通过添加 `%` 在你想匹配部分的左侧或右侧以示意匹配规则。以下是匹配**粗体**部分的一些示例：

- `like Bistr%` 可以匹配到“**Bistr**itz”（但不匹配“bistritz”，因为 `like`），
- `ilike '%IsTRiT%'` 可以匹配到“B**istrit**z”，
- `like %athan Harker` 可以匹配到 “Jon**athan Harker**”，
- `ilike %n h%` 可以匹配到“Jonatha**n H**arker”。

现在，让我们用 `filter` 过滤出所有首字母是大写字母 B 的城市。这意味着我们需要使用 `like`，因为它是对大小写敏感的：

```edgeql
select City {
  name,
  modern_name,
} filter .name like 'B%';
```

输出结果为：

```
{
  default::City {name: 'Buda-Pesth', modern_name: 'Budapest'},
  default::City {name: 'Bistritz', modern_name: 'Bistrița'},
}
```

你也可以用 `[]` 方括号索引一个字符串，索引从 0 开始计数。比如，字符串“Jonathan”的索引如下所示：

```
J o n a t h a n
0 1 2 3 4 5 6 7
```

因此 `'Jonathan'[0]` 是“J”，`'Jonathan'[4]` 是“t”。

现在，让我们试一下这个：

```edgeql
select City {
  name,
  modern_name,
} filter .name[0] = 'B'; # First character must be 'B'
```

同样，我们会得到我们想要的结果。不过要小心：如果你将数字设置得太高（超过字符串本身的长度），那么它会尝试在字符串之外进行搜索，这会带来错误。比如，如果我们将 0 更改为 18 (`filter .name[18] = 'B';`)，我们将得到：

```
ERROR: InvalidValueError: string index 18 is out of bounds
```

此外，如果你有一个名字为 `''` 的 `City`，即使搜索索引为 0 也会导致错误。

你还可以通过“切片（slice）”来获得字符串的一部分。例如：从 0 开始标记“Jonathan”，索引值如下所示：

```
|J|o|n|a|t|h|a|n|
0 1 2 3 4 5 6 7 8
```

“Jonathan”的长度是 8 个字符，因此它完全介于 0 和 8 之间。如果你在索引 2 和 5 之间“slice”它，你会得到“nat”（`'Jonathan'[2:5]` = 'nat'），因为它开始于 2，直到 5——但并不包括索引 5 对应的字符。

负的索引值从“Jonathan”的末尾开始计数，即从 8 开始，所以 -1 对应的是 `8 - 1` (= 7)，以此类推。

那么，如果你想确保不会因索引号数字过高而引发错误，该怎么办？使用 `like` 或 `ilike`，因为即使操作于空参数，它也只是会返回一个空集：`{}` 而不会是错误。因此，如果属性中有可能包含太短的数据，`like` 和 `ilike` 比使用索引更保险。这里还需要强调下：

- 在 Edgedb 中，“无数据”会被显示为空集：`{}`；
- `""`（一个空字符串）实际上也是数据。

记住它们有助于你理解他们两者之间的行为。


最后，你是否注意到我们刚刚用 `#` 写了一个注释？EdgeDB 中的注释很简单：一行中 `#` 右侧的任何内容都会在程序执行时被忽略，被视为“注释”。

因此语句：

```edgeql
select 1893#0503 is the first day of the book Dracula when...
;
```

只是会返回 `{1893}`.

[→ 点击这里查看到第 2 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 使用类型转换修改语句 `select '99' + '1'`，使其输出结果为 `{100}`；
2. 选择出所有以“Mu”开头的 `City`（需要区分大小写）；
3. 选择出所有 `NPC` 名字的第三个字母（即索引号为 2）；
4. 假设有一个抽象类型叫做 `HasAString`:

   ```sdl
   abstract type HasAString {
     string: str
   };
   ```

   你将如何修改 `Person` 类型成为 `HasAString` 的扩展类型？

5. 下面的查询仅会显示造访过的地方的 id。请问如何显示它们的名字？

   ```edgeql
   select Person {
     places_visited
   };
   ```

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森坐上马车，前往寒冷的山脉。_
