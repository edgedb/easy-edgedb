---
tags: Overloading Functions, Coalescing
---

# 第十二章 - 情况越来越糟糕

这一章并没有给我们的主人公们带来什么好消息。

> 只要有人不听范海辛（Van Helsing）的话，打开了露西房间的窗户，德古拉（Dracula）就会趁机潜入，每次男士们都需要通过献血来挽救露西。德古拉总是化成一团云溜进房间，吸露西的血，并在天亮之前溜走。露西变得十分虚弱，以至于她还活着都令人感到惊讶。与此同时，伦菲尔德（Renfield）冲出了牢房，他用刀袭击并割伤了苏厄德医生（Dr. Seward），当他看到苏厄德医生的血时，他停止了攻击并想要去喝他的血，他嘴里重复着：“血就是命！血就是命！”。收容所的安保带走了伦菲尔德，苏厄德医生感到十分困惑并试图找出原因。他认为这和其他事件可能存在某种联系。当天晚上，德古拉操控一匹狼打破了露西房间的窗户，并再次潜入……

但是对我们来说有个好消息，那就是我们将继续在本章学习笛卡尔乘积，以及如何重载一个函数。

## 重载函数

上一章，我们对一些角色使用了 `fight()` 函数，但大多数角色的 `strength` 是 `{}`。这就是为什么客栈老板（the Innkeeper）可以打败德古拉（Dracula），但这显然是不可能发生的事情。

乔纳森·哈克（Jonathan Harker）是一个人类，但他十分强壮，因此我们将他的力量值设为 5。除了有点独特（感觉又像人类又像吸血鬼）的伦菲尔德（Renfield），我们将 5 视为人类力量的最大值，所以其他人的力量都应该在 1 到 5 之间。EdgeDB 有一个名为 {eql:func}`docs:std::random` 的随机函数，它会给出一个介于 0.0 和 1.0 之间的 `float64`。在此，我们还会使用到一个叫做 {eql:func}`docs:std::round` 的函数，它可以对数字进行四舍五入，最后我们将结果转换为 `<int16>`，即如下所示：

```edgeql
SELECT <int16>round(random() * 5);
```

那么，现在我们将使用它来更新所有 `strength` 尚为 `{}` 的 `Person`，并赋予它们一个随机的强度值。

```edgeql
UPDATE Person
  FILTER NOT EXISTS .strength
  SET {
    strength := <int16>round(random() * 5)
};
```

接着我们需要确保德古拉伯爵获得 20 点力量，因为他是德古拉呀：

```edgeql
UPDATE Vampire
FILTER .name = 'Count Dracula'
SET {
  strength := 20
};
```

现在让我们通过执行 `SELECT Person.strength;` 来看看上面的操作是否有效：

```
{3, 3, 3, 2, 3, 2, 2, 2, 3, 3, 3, 3, 4, 1, 5, 10, 4, 4, 20, 4, 4, 4, 4}
```

看起来它奏效了！

现在，让我们来对函数 `fight()` 进行重载。目前 `fight()` 只适用于一个 `Person` 对战另一个 `Person`，但在书中所有角色将聚集在一起并试图合力击败德古拉。因此，我们需要重载该函数，以便支持多个角色并肩战斗的情形。这里有很多方法可以做到，但我们会选择一个简单的：

```sdl
function fight(names: str, one: int16, two: str) -> str
  using (
    WITH opponent := assert_single((SELECT Person FILTER .name = two))
    SELECT
        names ++ ' win!' IF one > opponent.strength ELSE
        two ++ ' wins!'
  );
```

第一参数是用于接收并肩作战的人们的名字，第二个是用于接收他们的力量总和，第三个参数是用于接收人们要一起对抗的角色。这里需要注意，重载只在函数签名不同的情况下有效。下面是 `fight()` 函数现有的两个签名，我们可以对比一下：

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: str) -> str
```

如果我们试图用 `(Person, Person)` 作为输入重载 `fight()`，则重载并不会生效，因为两个签名是相同的。EdgeDB 是通过我们的输入来判断该调用哪种形式的函数的。

所以，现在虽然是调用了同名函数，但我们的输入是并肩作战的人们的名字，及他们的力量和，然后是他们正在共同对抗的 `Person`。

现在乔纳森（Jonathan）和伦菲尔德（Renfield）将要尝试一起对抗德古拉（Dracula）。祝他们好运！

```edgeql
WITH
  jon_and_ren_strength := <int16>(
    SELECT sum(
      (SELECT NPC FILTER .name IN {'Jonathan Harker', 'Renfield'}).strength
    )
  ),
SELECT fight('Jon and Ren', jon_and_ren_strength, 'Count Dracula');
```

结果还是德古拉赢了：

```
{'Count Dracula wins!'}
```

他们输了，那如果是四个人呢？

```edgeql
WITH
  four_people_strength := <int16>(
    SELECT sum(
      (
        SELECT NPC
        FILTER .name IN {'Jonathan Harker', 'Renfield', 'Arthur Holmwood', 'The innkeeper'}
      ).strength
    )
  ),
SELECT fight('The four people', four_people_strength, 'Count Dracula');
```

翻转了！

```
{'The four people win!'}
```

这就是函数重载的工作原理——只要签名不同，你就可以创建具有相同名称的函数。

你也会在许多现有函数中看到重载的应用，例如 {eql:func}`docs:std::sum` 可以接收所有数字类型并返回其总和。{eql:func}`docs:std::to_datetime` 有着更有意思的重载，它支持各种输入来创建一个 `datetime`。

`fight()` 制作起来很有趣，但这种函数更适合在游戏中使用。因此，现在让我们来创建一个我们实际上更可能会用到的函数。由于 EdgeQL 是一种查询语言，所以最有用的函数通常是使查询变得更短的函数。

这是一个简单的函数，它告诉我们一个 `Person` 类型的对象是否造访了一个 `Place`：

```sdl
function visited(person: str, city: str) -> bool
  using (
    WITH person := (SELECT Person FILTER .name = person),
    SELECT city IN person.places_visited.name
  );
```

现在，我们的查询变得方便得多了：

```edgeql-repl
edgedb> SELECT visited('Mina Murray', 'London');
{true}
edgedb> SELECT visited('Mina Murray', 'Bistritz');
{false}
```

多亏了这个函数，即使是更复杂的查询也仍然具有相当的可读性：

```edgeql
SELECT(
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' ++ <str>visited('Jonathan Harker', 'Romania')
);
```

输出结果是：`{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`。

有关创建函数的文档可以查询 {ref}`这里 <docs:ref_eql_ddl_functions>`。你会发现我们可以使用 SDL 或 DDL 来创建函数，且两者之间没有太大区别。事实上，它们十分相似，唯一的区别是 DDL 需要使用 `CREATE`。换句话说，只需添加 `CREATE` 即可创建函数，而无需进行显式迁移（migration）。例如，下面是一个打招呼的函数：

```sdl
function say_hi() -> str
  using ('hi');
```

如果你想立即创建它，只需执行以下操作：

```edgeql
CREATE FUNCTION say_hi() -> str
  USING ('hi');
```

（关键字也可以用小写字母，没关系的，但我们提倡用大写字母）

当你执行 `DESCRIBE FUNCTION say_hi` 时，你会得到或多或少有些相同的内容：

```
{'CREATE FUNCTION default::say_hi() ->  std::str USING (\'hi\');'}
```

## 删除函数

你可以使用关键字 `DROP` 和函数签名来删除函数。不过，你只需指定输入，因为 EdgeDB 在识别一个函数时只查看输入。所以在我们的两个 `fight()` 函数的例子中：

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: str) -> str
```

你可以用 `DROP Fight(one: Person, two: Person)` 和 `DROP Fight(names: str, one: int16, two: str)` 来分别删除它们，并不需要包含 `-> str` 部分。

## 更多关于笛卡尔积

现在让我们来更多地了解 EdgeDB 中的笛卡尔乘积。你可能会惊讶地发现，即使只有一个输入为 `{}`，也总是会导致输出 `{}`，但这就是笛卡尔乘积的工作方式。请记住，`{}` 的长度为 0，任何乘以 0 的值也是 0。例如，让我们尝试将以 b 开头的地名和以 f 开头的地名加在一起。

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name;
```

输出是……，如果你没有阅读上一段，结果可能会令你感到出乎意料：

```
{}
```

是的！是一个空集。虽然我们并没有以 f 开头的地方，但是搜索以 b 开头的地方，我们明明会得到 `{'Buda-Pesth', 'Bistritz'}`。那么让我们直接用 `++` 连接 `{'Buda-Pesth', 'Bistritz'}` 和 `{}`，看看是否会是同样的效果。

```edgeql
SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
```

你可能以为会同样得到 `{}`。但实际上结果是……

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast or a conversion function.
```

这是另一个可能的出乎意料！不过这里有一个很重要的知识点：EdgeDB 需要我们对空集进行类型转换（指定），EdgeDB 是不会尝试去猜测这个空集是什么类型。因为如果只是一个 `{}`，我们是没有办法猜测出其类型的（编写者意图的），因此 EdgeDB 也不会尝试去猜测。你可能会猜到数组构造器也是如此，因此 `SELECT [];` 会返回错误：`QueryError: expression returns value of indeterminate type`。

好的，让我们再试一次，这次确保 `{}` 空集是 `str` 类型：

```edgeql-repl
edgedb> SELECT {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
```

好的，这下我们亲自动手确认了：`{}` 与另一个非空集合的连接（级联）总是返回 `{}`。但是如果想要达到以下两点，我们该怎么做呢？

- 如果两个集合均存在，则连接它们，
- 如果有一个是空集，则返回我们拥有的。

换言之，将 `{'Buda-Peth', 'Bistritz'}` 与另一个集合相加时，如何在另一个集合是空时返回原本的 `{'Buda-Peth', 'Bistritz'}`？

为此，我们可以使用合并操作符 {eql:op}`coalescing operator <docs:coalesce>`，写为 `??`。所以对于 `A ?? B` 可以很好地、简单地解释为：

`Evaluate to A for non-empty A, otherwise evaluate to B.`

即如果左侧（A）的项目不为空，它将返回该项目，否则将返回右侧（B）的项目。

这是一个快速示例：

```edgeql-repl
edgedb> SELECT <str>{} ?? 'Count Dracula is now in Whitby';
```

因为我们使用了 `??` 而不是 `++`，所以结果是 `{'Count Dracula is now in Whitby'}` 而不是 `{}`。

让我们回到最初的查询，这次使用合并运算符（coalescing operator）：

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     f_places := (SELECT Place FILTER .name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name
  IF EXISTS b_places.name AND EXISTS f_places.name
  ELSE b_places.name ?? f_places.name;
```

得到返回：

```
{'Buda-Pesth', 'Bistritz'}
```

这个结果更好。

但回到笛卡尔乘积。别忘了，当我们相加或连接集合时，我们要分别处理 _每个集合中的每个项目_。因此，如果我们将查询更改为分别搜索以 b（Buda-Pesth 和 Bistritz）和 m（Munich）开头的地方：

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER .name ILIKE 'm%'),
SELECT b_places.name ++ ' ' ++ m_places.name
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

我们得到的结果是：

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

而不是 `{'Buda-Pesth, Bistritz, Munich'}`。

现在，让我们在进一步解决问题的同时，引入名为 `array_agg` 和 `array_join` 的两个新函数。以下是他们的功能说明：

- {eql:func}`docs:std::array_agg` 将集合转换为数组（“聚合”它们）；
- {eql:func}`docs:std::array_join` 将数组转换为由指定分隔符间隔的单个字符串。现在来让我们试一下这个：

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER .name ILIKE 'm%'),
SELECT array_join(array_agg(b_places.name), ', ') ++ ', ' ++
  array_join(array_agg(m_places.name), ', ')
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

看起来还不错：输出是 `{'Buda-Pesth, Bistritz, Munich'}`。但是有一个小问题：

- 如果两个集合都不为空，我们得到的是一个带逗号的字符串，
- 如果有一个为空，我们得到的是一个字符串的集合。

这不是很健壮，因为输出没有保证一致性。此外，现在的查询似乎有点难以阅读。

最好的方法实际上也是最简单的：只需要使用关键字 `UNION`。

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER .name ILIKE 'm%'),
     both_places := b_places UNION m_places,
SELECT both_places.name;
```

最终，输出是：`{'Buda-Pesth', 'Bistritz', 'Munich'}`

现在有了这个更强大的查询，我们就可以在任何情况下使用它了，而不需要担心因为选了像 x 这样的字母作为过滤条件，导致匹配不到城市而最终得到 `{}`。下面，让我们看看所有包含 k 或 e 的地方：

```edgeql
WITH has_k := (SELECT Place FILTER .name ILIKE '%k%'),
     has_e := (SELECT Place FILTER .name ILIKE '%e%'),
     has_either := has_k UNION has_e,
SELECT has_either.name;
```

输出结果是:

```
{'Slovakia', 'Buda-Pesth', 'Castle Dracula'}
```

类似的，当对两个集合使用 `=` 或 `!=` 时，如果两侧均不为空集时，则其中一个集合的每一项都将与另一个集合的每一项依次进行比对；如果你认为有一侧可能为空集，则在进行比较时可以使用 `?=` 和 `?!=` 分别代替 `=` 和 `!=`，以避免最终得到的是没有意义的空集。比如，你可以做这样的查询：

```edgeql
WITH cities1 := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     cities2 := <str>{}, # Don't forget to cast to <str>
SELECT cities1 ?= cities2;
```

输出结果是：

```
{false, false, false}
```

而不是 `{}`。此外，如果你使用 `?=`，则两个空集被视为相等。你可以通过执行 `SELECT <str>{} = <str>{};` 和 `SELECT <str>{} ?= <str>{};` 来查看区别。所以：

```edgeql
SELECT Vampire.lover.name ?= Crewman.name;
```

将返回 `{true}`。（因为德古拉没有情人，船员也没有名字，所以双方返回的都是类型为 `str` 的空集。使用 `SELECT <str>{} ?= <str>{};` 则返回 `{true}`；使用 `SELECT <str>{} = <str>{};` 则返回 `{}`。）

[→ 点击这里查看到第 12 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 考虑下面这两个函数。EdgeDB 会接受第二个吗？

   第一个函数：

   ```sdl
   function gives_number(input: int64) -> int64
     using(input);
   ```

   第二个函数：

   ```sdl
   function gives_number(input: int64) -> int32
     using(<int32>input);
   ```

2. 那么下面两个函数呢？EdgeDB 会接受第二个吗？

   第一个函数：

   ```sdl
   function make64(input: int16) -> int64
     using(input);
   ```

   第二个函数：

   ```sdl
   function make64(input: int32) -> int64
     using(input);
   ```

3. `SELECT {} ?? {3, 4} ?? {5, 6};` 能工作吗？

4. `SELECT <int64>{} ?? <int64>{} ?? {1, 2}` 能工作吗

5. `SELECT array_join(array_agg(Person.name));` 在尝试获得一个含有所有人姓名的字符串，但它不能工作，问题出在哪里？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _其中一位男士试图通过献血来拯救露西。但这就够了吗？_
