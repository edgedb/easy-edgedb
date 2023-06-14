# Chapter 14 Questions and Answers

#### 1. 如何仅显示所有 `Person` 对象的编号？比如，如果有 20 个，则显示 `1, 2, 3..., 18, 19, 20`。

一种方法是使用 `enumerate()`。如果我们只是从 0 开始显示，就容易。`enumerate()` 会为输入集合中的每个元素都输出一个包含两个项目的元组。第一个则是 `int64` 类型的索引，所以我们选择使用 `enumerate()`：

```edgeql
select enumerate(Person).0;
```

结果显示出：`{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19}`

注意：这里选择 `.1` 的话，会产生一堆没有细节的对象。

回到问题本身，如果要显示从 1 开始的数字，我们只需在使用 `enumerate` 时加 1：

```edgeql
select enumerate(Person).0 + 1
```

#### 2. 使用一个可计算的反向链接，如何显示 1）所有名称中带有 `o` 的 `Place` 的对象（及他们的名称）；2）访问过这些地方的人物的名字？

如果你一步一步开始，则并不太难。首先用一个过滤器来获取所有名字满足条件的 `Place` 对象：

```edgeql
select Place {
  name
} filter .name like '%o%';
```

结果是：

```edgeql
{
  default::Country {name: 'Romania'},
  default::Country {name: 'Slovakia'},
  default::City {name: 'London'},
}
```

然后，我们将可计算的反向链接添加到同一个查询，并赋值给计算属性 `visitors`：

```edgeql
select Place {
  name,
  visitors := .<places_visited[is Person].name
} filter .name like '%o%';
```

于是，我们就可以看到都有谁分别到访过这些地方了：

```
{
  default::Country {name: 'Romania', visitors: {'Jonathan Harker', 'Count Dracula'}},
  default::Country {name: 'Slovakia', visitors: {}},
  default::City {
    name: 'London',
    visitors: {
      'Jonathan Harker',
      'The innkeeper',
      'Mina Murray',
      'Lucy Westenra',
      'John Seward',
      'Quincey Morris',
      'Arthur Holmwood',
      'Renfield',
      'Abraham Van Helsing',
    },
  },
}
```

伦敦作为访问量最大的地方明显胜出！如果你想，你也可以添加一个 `visitor_numbers := count(.<places_visited[is Person].name)` 来获取访问者的数量。

#### 3. 使用一个可计算的反向链接，如何显示所有后来成为了 `MinorVampire` 的 `Person` 对象？

我们可以再次通过使用一个计算（computed）链接来做到这一点，我们将其称为 `later_vampire`。然后我们使用一个可计算的反向链接指回 `MinorVampire`，即通过链接 `former_self` 链接到 `Person` 的 `MinorVampire`：

```edgeql
select Person {
  name,
  later_vampire := .<former_self[is MinorVampire].name
} filter exists .later_vampire;
```

结果中只有露西：

`{default::NPC {name: 'Lucy Westenra', later_vampire: {'Lucy'}}}`

看起来还不错，但我们还可以做得更好——这里的 `later_vampire` 没有告诉我们关于它的类型。那么，让我们为其添加一些类型信息：

```edgeql
select Person {
  name,
  later_vampire := .<former_self[is MinorVampire] {
    name,
    __type__: {
      name
    }
  }
} filter exists .later_vampire;
```

现在，我们可以从结果中同时看到 `later_vampire` 的类型是 `MinorVampire`：

```
{
  default::NPC {
    name: 'Lucy Westenra',
    later_vampire: {
      default::MinorVampire {
        name: 'Lucy',
        __type__: schema::ObjectType {name: 'default::MinorVampire'},
      },
    },
  },
}
```

#### 4. 如何给 `MinorVampire` 类型一个名为 `note` 的注解，并添加说明 `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`？

首先，你得创建这个注释，因为它还不存在：

```sdl
abstract annotation note;
```

然后，你只需将其放入 `MinorVampire` 类型就可以了！

```sdl
type MinorVampire extending Person {
  former_self: Person;
  annotation note := 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type';
}
```

#### 5. 如何在查询中看到 `MinorVampire` 的 `note` 注释？

如下所示：

```edgeql
select (introspect MinorVampire) {
  name,
  annotations: {
    name,
    @value
  }
};
```

这是输出：

```
{
  schema::ObjectType {
    name: 'default::MinorVampire',
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type',
      },
    },
  },
}
```

当然，如果该类型存在多个注释，比如 `title`、`description`，这里也同样会输出显示。
