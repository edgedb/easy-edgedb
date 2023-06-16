# Chapter 9 Questions and Answers

#### 1. 为什么下面这个插入不起作用，该如何修复？

它需要是一个集合而不是一个数组，所以将 `IN` 后面的括号改为 `{}`：

```edgeql
for castle in {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
union (
  insert Castle {
    name := castle
  }
);
```

#### 2. 如何在显示城堡名称的同时进行与上题相同的插入？

如下所示：

```edgeql
select (
  for castle in {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
  union (
    insert Castle {
      name := castle
    }
  )
) { name };
```

#### 3. 如果所有的吸血鬼都需要一个最小为 10 的力量值，如何修改 `Vampire` 类型？

由于 `strength` 来自 `abstract type Person`，你需要在 `Vampire` 里重载（overload）它并给它一个约束。`Vampire` 类型应被修改为：

```sdl
type Vampire extending Person {
  multi slaves: MinorVampire;
  overloaded strength: int16 {
    constraint min_value(10)
  }
}
```

#### 4. 如何更新所有的 `Person` 类型的对象，表明他们都死于 1893 年 9 月 11 日？

如下所示，你将对每一个 `Person` 的 `last_appearance` 进行更新：

```edgeql
update Person
set {
  last_appearance := <cal::local_date>'1893-09-11'
};
```

#### 5. 如果所有名字中带有 `e` 或 `a` 的 `Person` 角色都被复活了，你将如何更新？

你可以通过在集合上而不是在单个字母上使用 `like` 来进行 `update`：

```edgeql
update Person filter .name like {'%a%', '%e%'}
set {
  last_appearance := {}
};
```

如果为了确保成功你想同时显示执行结果，也可以这样写：

```edgeql
select (
  update Person filter .name ilike {'%a%', '%e%'}
  set {
    last_appearance := {}
  }
) {
  name,
  last_appearance
};
```
