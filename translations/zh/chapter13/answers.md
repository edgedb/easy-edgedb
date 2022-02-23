# Chapter 13 Questions and Answers

#### 1. 如何用一个单独的插入语句插入一个名为“Mr. Swales”，且到访过名为“York”的 `City`，名为“England”的 `Country` 以及名为“Whitby Abbey”的 `OtherPlace` 的 `NPC`？

类似于我们在之前的章节中插入 `Ship` 时所做的：

```edgeql
INSERT NPC {
  name := 'Mr. Swales',
  places_visited := {
    (INSERT City {
      name := 'York'
    }),
    (INSERT Country {
      name := 'England'
    }),
    (INSERT OtherPlace {
      name := 'Whitby Abbey'
    }),
  }
};
```

#### 2. 下面这个内省查询结果的可读性如何？

这个查询：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links
};
```

三分之一可读：只有 `name` 部分会显示为人类可读的名称。结果如下：

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

在两个地方添加 `: {name}` 则可使其完全可读：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {name},
  links: {name},
};
```

结果是：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'name'},
    },
    links: {
      schema::Link {name: '__type__'},
      schema::Link {name: 'crew'},
      schema::Link {name: 'sailors'},
    },
  },
}
```

#### 3. 查看 `Vampire` 类型有哪些链接的最简单的方法是什么？

类似于 `SELECT Vampire.name` 可以给出所有 `Vampire` 对象的名称，你可以这样做：

```edgeql
SELECT (Introspect Vampire).links { name };
```

输出是：

```
{
  schema::Link {name: 'slaves'},
  schema::Link {name: 'lover'},
  schema::Link {name: '__type__'},
  schema::Link {name: 'places_visited'},
}
```

#### 4. `SELECT DISTINCT {1, 2} + {1, 2};` 的输出会是什么？

输出是：

```
{2, 3, 3, 4}
```

你可以看到 `DISTINCT` 是独立作用于一个集合的（所以这里只是作用在了第一个上），所以 `SELECT DISTINCT {1, 2} + {1, 2};` 和 `SELECT {1, 2} + {1, 2};` 是相同的。如果你要写 `SELECT DISTINCT {2, 2}`，输出将只是 `{2}`。

#### 5. `SELECT DISTINCT {2, 2} + {2, 2};` 的输出会是什么？

输出将为 `{4, 4}`，因为 `DISTINCT` 仅作用于第一个集合。

要想输出为 `{4}`，你可以重复使用 `DISTINCT`：即 `SELECT DISTINCT {2, 2} + DISTINCT {2, 2};`。或者你可以像这样包装整个运算：`SELECT DISTINCT({2, 2} + {2,2})`。
