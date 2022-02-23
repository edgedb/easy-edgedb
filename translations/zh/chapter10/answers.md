# Chapter 10 Questions and Answers

#### 1. 尝试通过一个插入语句插入两个 `NPC` 类型的对象，其中包含 `name`, `first_appearance` 和 `last_appearance` 信息。

如下所示：

```edgeql
FOR person IN {('Jimmy the Bartender', '1887-09-10', '1887-09-11'), ('Some friend of Jonathan Harker', '1887-07-08', '1887-07-09')}
UNION (
  INSERT NPC {
    name := person.0,
    first_appearance := <cal::local_date>person.1,
    last_appearance := <cal::local_date>person.2
  }
);
```

#### 2. 这里还有两个要插入的 `NPC`，后者的最后有一个空集（因为她还没有死）。插入它们时，我们会遇到什么问题？

问题是 EdgeDB 不知道后者的最后一个 `{}` 是什么类型。你可以通过快速执行一下 `SELECT {('Dracula\'s Castlevisitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05 -08', {})}`：

发现结果：

```
ERROR: QueryError: operator 'UNION' cannot be applied to operands of type 'tuple<std::str, std::str, std::str>' and 'tuple<std::str, std::str, anytype>'
  Hint: Consider using an explicit type cast or a conversion function.
```

类型转换为 `<str>{}` 是一种选择，但我们可以做一些更健壮的事情。首先将 `{}` 更改为空字符串，然后指定 `last_appearance` 的长度必须为 10，否则将其设为 `<cal::local_date>{}`：

```edgeql
FOR person IN {('Dracula\'s Castle visitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05-08', '')}
UNION(
  INSERT NPC {
    name := person.0,
    first_appearance := <cal::local_date>person.1 IF len(person.1) = 10 ELSE <cal::local_date>{},
    last_appearance := <cal::local_date>person.2 IF len(person.2) = 10 ELSE <cal::local_date>{}
  }
);
```

#### 3. 你将如何按人物姓名的最后一个字母对 `Person` 类型的对象进行排序？

只需通过切片得到最后一个字母并进行排序：

```edgeql
SELECT Person {
  name
} ORDER BY .name[-1];
```

#### 4. 尝试插入一个名为 `''` 的 `NPC`。现在，你将如何在问题 3 中执行相同的查询？

如果我们尝试执行与上一题相同的查询语句，则会收到以下错误：

```
ERROR: InvalidValueError: string index -1 is out of bounds
```

不过我们可以调整。一种方法是过滤掉名字长度小于 1 的人物：

```edgeql
SELECT Person {
  name
} FILTER len(.name) > 0 ORDER BY .name[-1];
```

或者，如果你并不想过滤掉它，你可以为长度不足 1 的名字添加一个虚拟字符，然后再进行排序：

```edgeql
SELECT Person {
  name := ' ' ++ .name IF len(.name) = 0 ELSE .name
} ORDER BY .name[-1];
```

#### 5. 范海辛医生（Dr. Van Helsing ）有一份 `MinorVampire` 的名单，上面有他们的名字和力量。我们的数据库中已经有一些 `MinorVampire` 了。如果对象已经存在，你将如何在确保 `UPDATE` 的同时 `INSERT` 不存在的？ 

你可以用 `UNLESS CONFLICT ON` 和 `ELSE` 来做到这一点。比如，一个名为 Carmilla 的插入将如下所示：

```edgeql
INSERT MinorVampire {
  name := 'Carmilla',
  strength := 10,
} UNLESS CONFLICT ON .name
ELSE (
  UPDATE MinorVampire
  SET {
    name := 'Carmilla',
    strength := 10,
  }
);
```

然后即使他的数据有一个冲突的名字（比如 'Woman 1'，在我们的数据库里已经存在），它仍然会正确地更新“力度”，而不是简单地放弃：

```edgeql
INSERT MinorVampire {
  name := 'Woman 1',
  strength := 7,
} UNLESS CONFLICT ON .name
ELSE (
  UPDATE MinorVampire
  SET {
    name := 'Woman 1',
    strength := 7,
  }
);
```
