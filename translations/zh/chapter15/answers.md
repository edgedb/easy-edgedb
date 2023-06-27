# Chapter 15 Questions and Answers

#### 1. 如何创建一个名为“Horse”的类型，且其属性 `required name: str` 的值只能是“Horse”？

使用 `expression on` 则很容易，并用 `__subject__.name` 来引用我们要插入的对象的名称：

```sdl
type Horse {
  required name: str;
  constraint expression on (__subject__.name = 'Horse');
};
```

#### 2. 如何让用户们能了解到 `name` 需要被赋值“Horse”？如何提示他们？

像下面这样把提示放在 `constraint` 中：

```sdl
type Horse {
  required name: str;
  constraint expression on (__subject__.name = 'Horse') {
    errmessage := 'All Horses must be named \'Horse\''
  }
};
```

#### 3. 如何确保 `NPC` 类型的 `name` 长度始终在 5 到 30 个字符之间？

首先，这里是 `NPC` 类型的定义：

```sdl
type NPC extending Person {
  overloaded age: int16 {
    constraint max_value(120);
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

由于 `name` 来自 `Person`，我们可以重载它并进行约束。使用 `expression on` 和 `len()`，我们可以确保它始终大于 4 且小于 31：

```sdl
type NPC extending Person {
  overloaded name: str {
    constraint expression on (len(__subject__) > 4 and len(__subject__) < 31)
  }
  overloaded age: int16 {
    constraint max_value(120);
  }
  overloaded multi places_visited: Place {
    default := (select City filter .name = 'London');
  }
}
```

另一种选择是使用我们在本章中学到的约束：`max_len_value()` 和 `min_len_value()`。

#### 4. 如何创建一个名为 `display_coffins` 的函数来显示出所有棺材数量大于 0 的 `HasCoffins` 类型的对象？

方法如下所示：

```sdl
function display_coffins() -> set of HasCoffins
  using(
    select HasCoffins filter .coffins > 0
  );
```

请注意，`HasCoffins` 没有像 `name` 这样的属性，所以使用它进行查询时，应该像下面这样编写：

```edgeql
select display_coffins() {
  [is City].name,
  [is City].population,
};
```