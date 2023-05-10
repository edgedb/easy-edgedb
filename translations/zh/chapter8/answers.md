# Chapter 8 Questions and Answers

#### 1. 如何选择出所有 `Place` 和他们的名字，如果当它是个 `Castle` 时，同时显示属性 `doors`？

答案如下：

```edgeql
select Place {
  name,
  [is Castle].doors
};
```

没有 `[is Castle]`，将无法正常执行。

#### 2. 如何在选择 `Place` 时，用 `city_name` 显示当它是个 `City` 时的 `name`，并用 `country_name` 显示当它是个 `Country` 时的 `country_name`？

```edgeql
select Place {
  city_name := [is City].name,
  country_name := [is Country].name
};
```

#### 3. 基于上一题，如果只想显示属于 `City` 或 `Country` 类型的结果，该如何做？

由问题 2 可以得到如下的输出结果：

```
{
  default::City {city_name: 'Munich', country_name: {}},
  default::City {city_name: 'Buda-Pesth', country_name: {}},
  default::City {city_name: 'Bistritz', country_name: {}},
  default::City {city_name: 'London', country_name: {}},
  default::Castle {city_name: {}, country_name: {}},
  default::Country {city_name: {}, country_name: 'Hungary'},
  default::Country {city_name: {}, country_name: 'Romania'},
  default::Country {city_name: {}, country_name: 'France'},
  default::Country {city_name: {}, country_name: 'Slovakia'},
}
```

像 `Object {city_name: {}, country_name: {}},` 这样的结果对我们没有用，因此我们可以用 `exists` 将它们过滤掉： 

```edgeql
select Place {
  city_name := [is City].name,
  country_name := [is Country].name
} filter exists .city_name or exists .country_name;
```

另一种过滤方式是使用 `filter Place is City | Country`。你可能熟悉其他编程语言中的 `|`。在 EdgeDB 中，它们被称为类型联合运算符（the type union operator），你将在第 13 章中了解到更多相关信息。

#### 4. 如何显示所有没有 `lover` 的 `Person` 对象及其姓名和所属类型的名称？

要获得所有这些单身人士的姓名和所属对象类型，只需执行以下操作：

```edgeql
select Person {
  name,
  __type__: {
    name
  }
} filter not exists .lover;
```

别忘了 `__type__` 后面的 `name`！没有 `name` 也不会报错，但是类型名称将会显示成类似 `__type__: Object {id: 20ef52ae-1d97-11eb-8cb6-0de731b01cc9}` 这样，对我们并没有什么用处。

#### 5. 下面这个查询需要修复什么？提示：有两个地方是必须要修复的，还有一个地方应该可以修改得更具有可读性。

需要修复的两个部分是：1) `name` 后面加上 `,`，2) `[is Castle]` 后面加上 `.`：

```edgeql
select Place {
  __type__,
  name,
  [is Castle].doors
};
```

_应该_ 修复的部分是：我们应该在 `__type__` 后面添加 `{ name }`，以便我们可以从输出中读懂它的类型名称（而不是尝试读懂 `__type__: Object {id: e0a9ab38-1e6e-11eb-9497-5bb5357741af}`）。最终，代码修改为：

```edgeql
select Place {
  __type__: {
    name
  },
  name,
  [is Castle].doors
};
```
