# Chapter 11 Questions and Answers

#### 1. 编写一个名为 `get_lucies()` 的函数，使其只返回所有名称与“Lucy Westenra”匹配的 `Person` 对象？

这很简单，只是别忘记返回 `set of Person`：

```sdl
function get_lucies() -> set of Person
  using (
    select Person filter .name = 'Lucy Westenra'
  );
```

然后你可以按照如下方式使用它：

```edgeql
select get_lucies() {
  name,
  places_visited: {name}
};
```

因为在 `Person` 的 `name` 有个 `exclusive constraint`，只一个`NPC`可以有'Lucy Westenra'的名字。但是，这个constraint就是从abstract type `Person`让delegated的，就是说所有的extending Person的type可以有一个自己的'Lucy Westenra'这个名字。也就是说你可以插入一个叫'Lucy Westenra'的`Sailor`。然后用get_lucies()的话就返回两个叫Lucy Westenra的Person。

#### 2. 编写一个函数：接收两个字符串，并返回所有名称可以匹配输入的两个字符串中任意一个的 `Person` 对象？

让我们将函数命名为 `get_two()`。定义如下所示：

```sdl
function get_two(one: str, two: str) -> set of Person
  using (
    with person_1 := (select Person filter .name = one),
         person_2 := (select Person filter .name = two),
    select {person_1, person_2}
  );
```

下面我将用该函数对 John Seward、Count Dracula 和他们奴仆的姓名进行查询：

```edgeql
select get_two('John Seward', 'Count Dracula') {
  name,
  [is Vampire].slaves: {name},
};
```

这是输出结果：

```
{
  default::NPC {name: 'John Seward', slaves: {}},
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Woman 1'},
      default::MinorVampire {name: 'Woman 2'},
      default::MinorVampire {name: 'Woman 3'},
    },
  },
}
```

#### 3. 下面的语句将会输出什么？

看下输入，你就会知道将有 16 行信息被生成，因为每个集合的每个部分都会与其他每个集合的每个部分进行组合：

```edgeql
SELECT {'Jonathan', 'Arthur'} ++ {' loves '} ++ {'Mina', 'Lucy'} ++ {' but '} ++ {'Dracula', 'The inkeeper'} ++ {' doesn\'t love '} ++ {'Mina', 'Jonathan'};
```

(2 * 1 * 2 * 1 * 2 * 1 * 2 = 16)

下面是输出的结果：

```
{
  'Jonathan loves Mina but Dracula doesn\'t love Mina',
  'Jonathan loves Mina but Dracula doesn\'t love Jonathan',
  'Jonathan loves Mina but The inkeeper doesn\'t love Mina',
  'Jonathan loves Mina but The inkeeper doesn\'t love Jonathan',
  'Jonathan loves Lucy but Dracula doesn\'t love Mina',
  'Jonathan loves Lucy but Dracula doesn\'t love Jonathan',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Mina',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Mina but Dracula doesn\'t love Mina',
  'Arthur loves Mina but Dracula doesn\'t love Jonathan',
  'Arthur loves Mina but The inkeeper doesn\'t love Mina',
  'Arthur loves Mina but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Lucy but Dracula doesn\'t love Mina',
  'Arthur loves Lucy but Dracula doesn\'t love Jonathan',
  'Arthur loves Lucy but The inkeeper doesn\'t love Mina',
  'Arthur loves Lucy but The inkeeper doesn\'t love Jonathan',
}
```

#### 4. 如何制作一个函数来计算一个城市比另一个城市大多少倍？

这是一种方法：

```sdl
function two_cities(city_one: str, city_two: str) -> float64
  using (
    WITH first_city := (SELECT City FILTER .name = city_one),
         second_city := (SELECT City FILTER .name = city_two),
    SELECT first_city.population / second_city.population
  );
```

然后我们会像下面一样使用它：

```edgeql-repl
edgedb> SELECT two_cities('Munich', 'Bistritz');
{25.277252747252746}
edgedb> SELECT two_cities('Munich', 'London');
{0.06572085714285714}
```

参数 `city_one` 和 `city_two` 当然也可以定义为 `City` 类型，但是使用起来可能会有些别手。

#### 5. `SELECT (City.population + City.population)` 和 `SELECT ((SELECT City.population) + (SELECT City.population))` 会产生不同的结果吗？

> 第十章中我们定义过五个城市的人口，在这里列出，便于解题：
> * Buda-Pesth (Budapest): 402706
> * London: 3500000
> * Munich: 230023
> * Whitby: 14400
> * Bistritz (Bistrița): 9100

题中的两个语句会产生不同结果。第一个是单个集合上的 `SELECT`，会将每个项目添加到自身，结果是：`{28800, 460046, 805412, 18200, 7000000}`

同时，第二个是作用在两个集合上，其中每个项目都会分别和另一个集合里的每个项目进行一次相加。因此会得到 25（5 * 5）个结果：

```
{
  28800,
  244423,
  417106,
  23500,
  3514400,
  244423,
  460046,
  632729,
  239123,
  3730023,
  417106,
  632729,
  805412,
  411806,
  3902706,
  23500,
  239123,
  411806,
  18200,
  3509100,
  3514400,
  3730023,
  3902706,
  3509100,
  7000000,
}
```
