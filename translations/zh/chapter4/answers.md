# Chapter 4 Questions and Answers

#### 1. 下面的插入语句无法正常工作：

使用关键字 `detached` 可以使其正常工作，这样我们就可以从普通的 `NPC` 类型中进行提取，而不是从我们正在插入的 `NPC` 类型中提取：

```edgeql
insert NPC {
  name := 'I Love Mina',
  lover := assert_single(
    (select detached NPC filter .name like '%Mina%')
  )
};
```

另一种可行的方法是：

```edgeql
insert NPC {
  name := 'I Love Mina',
  lover := assert_single(
    (select Person filter .name like '%Mina%')
  )
};
```

我们将 `select NPC` 改为了 `select Person`。尽管 `NPC` 是从 `Person` 扩展而来的，但 `Person` 是一种不同的类型，因此我们可以从中提取而无需使用 `detached`。

#### 2. 如何显示名称里包含字母 `a` 的 `Person` 对象，且最多显示 2 个（以及它们的 `name` 属性）？

像下面这样，使用 `like` 和 `limit`：

```edgeql
select Person {
  name
} filter .name like '%a%' limit 2;
```

#### 3. 如何显示出从未访问过任何地方的 `Person` 对象（以及它们的 `name` 属性）？

使用 `not exists`：

```edgeql
select NPC {name} filter not exists .places_visited;
```

#### 4. 如何改为：时间里有 9 则显示 {true}，没有则显示 {false}？

原始的查询语句是：

```edgeql
select has_nine_in_it := <cal::local_time>'09:09:09';
```

使用 `<str>` 将 `cal::local_time` 转换为一个字符串，然后在结尾添加 `like '%9%'`，如下所示:

```edgeql
select has_nine_in_it := <str><cal::local_time>'09:09:09' like '%9%';
```

#### 5. 你将如何在插入的同时，用 `select` 来显示它的 `name`，`age` 和 `age_ten_years_later`？`age_ten_years_later` 是指 `age` 加 10。

原始的插入语句是：

```edgeql
insert NPC {
  name := "The Innkeeper's Son",
  age := 10
};
```

然后把它包装到小括号中，加上 `select` 部分并去掉插入语句结尾的 `;`：

```edgeql
select (
  insert NPC {
    name := "The Innkeeper's Son",
    age := 10
  }
);
```

现在只需像其他 `select` 一样添加上所需显示的字段，并添加上 `age_ten_years_later`：

```edgeql
select (
  insert NPC {
    name := "The Innkeeper's Son",
    age := 10
  }
) {
  name,
  age,
  age_ten_years_later := .age + 10
};
```

输出结果为：`{default::NPC {name: 'The Innkeeper\'s Son', age: 10, age_ten_years_later: 20}}`
