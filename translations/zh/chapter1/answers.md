# Chapter 1 Questions and Answers

#### 1. 输入下面的代码将会被返回错误，请尝试通过添加一个字符，使其返回结果为 `{true}`。

将 `WITH my_name = 'Timothy'` 修改为 `WITH my_name := 'Timothy'`。这样将给 `my_name` 赋值字符串 'Timothy'（而不是仅仅使用 '=' 尝试用 'Timothy' 和一个上不存在的变量 `my_name` 做比较）。然后用 '!=' 对被赋予 `'Timothy'` 的 `my_name` 与 `'Benjamin'` 进行比较，并返回了 `{true}`，因为他们确实不相等。

---

#### 2. 请尝试插入一个名为 Constantinople 的 `City`，其现代的命名为 İstanbul。

```edgeql
INSERT City {
  name := 'Constantinople',
  modern_name := 'İstanbul' # Comma after here is fine if you want
};
```

---

#### 3. 请尝试显示数据库中所有城市的名称。（提示：使用单行代码就可以做到，并不需要使用 `{}`）

`SELECT City.name;`

---

#### 4. 请尝试 SELECT 出所有城市的 `name` 和 `modern_name`，并将 `.name` 显示为 `old_name`，将 `.modern_name` 显示为 `name_now`。

```edgeql
SELECT City {
  old_name := .name,
  name_now := .modern_name,
};
```

当然，这并不意味着我们对这些属性的名称进行了修改。`old_name` 和 `name_now` 仅会在本次查询结果中出现。

---

#### 5. 键入 `SelecT City;` 会发生错误吗？

不会，因为 EdgeDB 里的关键字不区分大小写。
