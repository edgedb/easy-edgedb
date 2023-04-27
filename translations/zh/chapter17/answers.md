# Chapter 17 Questions and Answers

#### 1. 试试只用单行代码显示所有 NPC 的姓名、力量值、其到访城市的名称和该城人口，以及 NPC 年龄（如果年龄 = `{}`，则显示 0）。

这可以在一行上完成，但别忘了，我们需要使用 `[is City]`，因为 `places_visited` 链接的是 `Place`，但只有 `City` 有人口属性。答案如下所示：

```edgeql
select (
  NPC.name,
  NPC.strength,
  (NPC.places_visited[is City].name, NPC.places_visited[is City].population),
  NPC.age if exists NPC.age else 0
);
```

#### 2. 上题的查询结果中显示了很多没有上下文的数字，如何能使其变得更友好易读呢？

上题的查询的执行结果是：

```
{
  ('Mina Murray', 0, ('London', 3500000), 0),
  ('John Seward', 0, ('London', 3500000), 0),
  ('Quincey Morris', 0, ('London', 3500000), 0),
  # ...
}
```

我们可以在此运用“命名元组”：

```edgeql
select (
  name := NPC.name,
  strength := NPC.strength,
  city_populations := (NPC.places_visited[is City].name, NPC.places_visited[is City].population),
  age := NPC.age if exists NPC.age else 0
);
```

现在输出看起来好多了：

```
{
  (name := 'Mina Murray', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'John Seward', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'Quincey Morris', strength := 0, city_populations := ('London', 3500000), age := 0),
  # ...
}
```

如果需要，你甚至可以使用 `city_name` 和 `population` 命名元组内的元组。
