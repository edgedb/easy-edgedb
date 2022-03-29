# Chapter 7 Questions and Answers

#### 1. 如何选择出每一个 City 及他们名字的长度？

很容易，使用函数 `len()`：

```edgeql
SELECT City {
  name,
  name_length := len(.name)
};
```

#### 2. 如何选择出每一个 City 并展示其 `name` 长度减去 `modern_name` 长度的结果，如果 `modern_name` 不存在，则显示 0。 

为此，我们可以使用我们的老朋友 `IF EXISTS` 和 `ELSE`：

```edgeql
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 0
};
```

#### 3. 如果在上一题中想用 `'Modern name does not exist'` 替代 0，作为 `modern_name` 不存在时的结果显示，该如何做？

首先，如果像下面一样只是做简单的替换是无法工作的：

```edgeql
SELECT City {
  name,
  length_difference := len(.name) - len(.modern_name) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

你可能能猜到原因：`length_difference` 中的项目有时可能是 `int64`，而有时是 `str`，这是不可接受的。错误提示：

```
error: operator 'std::IF' cannot be applied to operands of type 'std::int64', 'std::bool', 'std::str'
```

因此，我们可以将结果统一转换为字符串：

```edgeql
SELECT City {
  name,
  length_difference := <str>(len(.name) - len(.modern_name)) IF EXISTS .modern_name ELSE 'Modern name does not exist'
};
```

结果如下所示：

```
{
  default::City {name: 'Munich', length_difference: 'Modern name does not exist'},
  default::City {name: 'Buda-Pesth', length_difference: '2'},
  default::City {name: 'Bistritz', length_difference: '0'},
  default::City {name: 'London', length_difference: 'Modern name does not exist'},
}
```

#### 4. 如果已经有 7 个 NPC 存在，你将如何插入名为“NPC number 8”的 NPC？

如下所示：

```edgeql
INSERT NPC {
  name := 'NPC number ' ++ <str>(count(DETACHED NPC) + 1)
};
```

`SELECT count(NPC)` 给出了 NPC 数量，但我们同时插入了一个 `NPC`，所以我们需要 `DETACHED` 来选择普遍的 `NPC` 类型对象。


#### 5. 如何选择出名字最短的 `Person` 类型的对象？

首先，下面是错误的写法：

```edgeql
SELECT Person {
  name,
} FILTER len(.name) = min(len(Person.name));
```

这似乎可行，但 `min(len(Person.name))` 是我们当前选择的 `Person` 的最小长度。这结果将是：每一个 `Person` 类型对象及他们的名字都会出现。

通过添加 `DETACHED` 来解决它：

```edgeql
SELECT Person {
  name,
} FILTER len(.name) = min(len(DETACHED Person.name));
```

我们也可以使用 `WITH` 来达到同样效果，这可能更加易读。这种方式中，我们不需要对 `WITH` 语句里的 `Person.name` 使用 `DETACHED`，因为我们在 `SELECT` 开始之前就完成了 `minimum_length` 的定义。

```edgeql
WITH minimum_length := min(len(Person.name))
SELECT Person {
  name,
} FILTER len(.name) = minimum_length;
```
