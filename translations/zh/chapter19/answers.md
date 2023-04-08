# Chapter 19 Questions and Answers

#### 1. 如何显示所有 `City` 的名称和它们所在 `Region` 的名称？

现在这对我们来说很简单，只需要一个可计算的反向链接。此外，我们还需要添加一个过滤器，过滤出位于“存在”的 `Region` 里的 `City` 对象：

```edgeql
SELECT City {
  name,
  region_name := .<cities[IS Region].name
} FILTER EXISTS .region_name;
```

#### 2. 基于上一题，如何显示所有 `City` 的名称和它们所在 `Region` 及 `Country` 的名称？

这是一个类似的查询，只是这次我们需要返回两个链接。

`.<cities[IS Region].name` 的意思是“通过链接 `cities` 链接到的 `Region` 的名称”，对 `Country` 可以使用同样的方式。如下所示：

`.<cities[IS Region].<regions[IS Country].name`

换句话说，是指“通过链接 `cities` 链接到的 `Region` 再通过链接 `regions` 链接到的 `Country` 的名称"。

```edgeql
SELECT City {
  name,
  region_name := .<cities[IS Region].name,
  country_name := .<cities[IS Region].<regions[IS Country].name
} FILTER EXISTS .country_name;
```

这为我们提供了从 `City` 到 `Country` 级别的漂亮输出：

```
{
  default::City {name: 'Dresden', region_name: {'Saxony'}, country_name: {'Germany'}},
  default::City {name: 'Leipzig', region_name: {'Saxony'}, country_name: {'Germany'}},
  default::City {name: 'Darmstadt', region_name: {'Hesse'}, country_name: {'Germany'}},
  default::City {name: 'Mainz', region_name: {'Hesse'}, country_name: {'Germany'}},
  default::City {name: 'Berlin', region_name: {'Prussia'}, country_name: {'Germany'}},
  default::City {name: 'Königsberg', region_name: {'Prussia'}, country_name: {'Germany'}},
}
```
