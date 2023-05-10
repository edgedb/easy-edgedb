# Chapter 16 Questions and Answers

#### 1. 如何找出名称是由两个单词构成的所有 `Person` 对象，并将其名称拆分成两个字符串？

你可以通过函数 `str_split()` 用空格进行分割，然后根据结果的长度进行过滤：

```edgeql
with two_names := (select str_split(Person.name, ' '))
select two_names filter len(two_names) = 2;
```

#### 2. 如何找出所有名称里有“ma”的 `Person` 对象，并显示出其名称？

这个查询很简单：

```edgeql
select (Person.name, find(Person.name, 'ma'));
```

请注意，第一个 `Person.name` 和第二个 `Person.name` 是相同的。但是，如果你将第二个更改为 `detached Person.name`，你将会因为“笛卡尔乘法”获得超过 100 个结果。

#### 3. 如何对 `Person` 类型的 `pen_name` 属性进行索引？

你可以直接通过使用 `index on (.pen_name)` 来实现。

也可以出于某种原因将它指定为像下面这样的表达式：

```sdl
index on (.name ++ ' ' ++ .degrees if exists .degrees else .name)
```

#### 4. 如何在展示所有 `Person` 对象的名称时，先以大写形式显示该名称，然后跟一个空格并以小写形式再显示一次其名称？

这里给出的可能是最简单的方法——只需使用 `str_upper` 和 `str_lower` 两个函数来完成，并使用 `++ ' '` 再它们中间添加一个空格：

```edgeql
select str_upper(Person.name) ++ ' ' ++ str_lower(Person.name);
```

#### 5. 如何使用 `re_match_all()` 来显示名称中带有 `Crewman` 的所有 `Person.name`？例如：Crewman 1，Crewman 2，等等。

你可以通过使用 `.` 通配符来匹配任何东西（至于使用几个通配符，取决于 `Crewman` 后还有几个字符位可以显示以及你想显示几位）：

```edgeql
select re_match_all('Crewman..', Person.name);
```
