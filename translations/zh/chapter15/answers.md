# Chapter 15 Questions and Answers

#### 1. 如何创建一个名为“Horse”的类型，且其属性 `required property name -> str` 的值只能是“Horse”？

使用 `expression on` 则很容易，并用 `__subject__.name` 来引用我们要插入的对象的名称：

```sdl
type Horse {
  required property name -> str;
  constraint expression on (__subject__.name = 'Horse');
};
```

#### 2. 如何让用户们能了解到 `name` 需要被赋值“Horse”？如何提示他们？

像下面这样把提示放在 `constraint` 中：

```sdl
type Horse {
  required property name -> str;
  constraint expression on (__subject__.name = 'Horse') {
    errmessage := 'All Horses must be named \'Horse\''
  }
};
```

#### 3. 如何确保 `NPC` 类型的 `name` 长度始终在 5 到 30 个字符之间？

首先，这里是 `NPC` 类型的定义：

```sdl
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

由于 `name` 来自 `Person`，我们可以重载它并进行约束。使用 `expression on` 和 `len()`，我们可以确保它始终大于 4 且小于 31：

```sdl
type NPC extending Person {
  overloaded property name {
    constraint expression on (len(__subject__) > 4 AND len(__subject__) < 31)
  }
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

另一种选择是使用我们在本章中学到的约束：`max_len_value()` 和 `min_len_value()`。

#### 4. 如何创建一个名为 `display_coffins` 的函数来显示出所有棺材数量大于 0 的 `HasCoffins` 类型的对象？

方法如下所示：

```sdl
function display_coffins() -> SET OF HasCoffins
  using(
    SELECT HasCoffins FILTER .coffins > 0
  );
```

请注意，`HasCoffins` 没有像 `name` 这样的属性，所以使用它进行查询时，应该像下面这样编写：

```edgeql
SELECT display_coffins() {
  [IS City].name,
  [IS City].population,
};
```

#### 5. 如何在不碰架构（schema）的情况下（不做”显式迁移“）创建上一题中的函数？

很简单，只需在它的最前面加上 `CREATE`！
