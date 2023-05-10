# Chapter 12 Questions and Answers

#### 1. 考虑下面这两个函数。EdgeDB 会允许同时定义它们吗？

不会接受，因为两个签名的输入部分是相同的：

```sdl
function gives_number(input: int64) -> int64
  using(input);

function gives_number(input: int64) -> int32
  using(<int32>input);
```

如果两者都可以接收 `int64`，那么在输入一个 `int64` 后，EdgeDB 将无法知道你要使用两者中的哪一个。

下面是错误信息：

```
error: cannot create the `default::gives_number(input: std::int64)` function: a function with the same signature is already defined
```

#### 2. 那么下面两个函数呢？EdgeDB 会允许同时定义它们吗？

会接受，因为可以接收的输入不同，所以它们的签名是不同的：一个接收 `int16`，另一个接收 `int32`。事实上，你可以通过使用 `describe` 来查看它们。例如，执行 `describe function make64 as text`：

```
{
  'function default::make64(input: std::int16) ->  std::int64 using (input);
   function default::make64(input: std::int32) ->  std::int64 using (input);',
}
```

但是请注意，执行 `select make64(8);` 实际上会产生错误！错误是：

```
error: function "make64(arg0: std::int64)" does not exist
  ┌─ query:1:8
  │
1 │ select make64(8);
  │        ^^^^^^^^^ Did you want one of the following functions instead:
default::make64(input: std::int16)
default::make64(input: std::int32)
```

这是因为 `select make64(8)` 输入的是一个 `int64`（**默认**），且它没有对应的签名。你需要在 `select make64(<int32>8);`（或 `<int16>8`）中使用类型转换以使其工作。

#### 3. `select {} ?? {3, 4} ?? {5, 6};` 能工作吗？

不能工作，因为 EdgeDB 不知道 `{}` 是什么类型。但是，如果你将 `{}` 转换为 `int64`：

```edgeql
select <int64>{} ?? {3, 4} ?? {5, 6};
```

它会给出输出：`{3, 4}`。

#### 4. `select <int64>{} ?? <int64>{} ?? {1, 2};` 能工作吗

可以工作，输出是 `{1, 2}`。如果你使用更多 `??`，它会一直运行，直到碰到不为空的东西。

知道了这一点，你大概可以猜到下面这个的输出结果：

```edgeql
select <int64>{} ?? <int64>{} ?? {1} ?? <int64>{} ?? {5};
```

输出是 `{1}`，它是第一个遇见的非空集。

#### 5. `select array_join(array_agg(Person.name));` 在尝试获得一个含有所有人姓名的字符串，但它不能工作，问题出在哪里？

错误信息给出了一个提示：

`error: function "array_join(arg0: array<std::str>)" does not exist`

这意味着 `array_join` 收到的输入无法匹配到任何函数签名。如果你查看了它的 {eql:func}`函数签名 <docs:std::array_join>`，你就会明白为什么了：它需要一个字符串作为第二个参数：

```sdl
std::array_join(array: array<str>, delimiter: str) -> str
```

因此，我们可以将其更改为 `select array_join(array_agg(Person.name), ' ');` 或 `select array_join(array_agg(Person.name), ' is awesome ');` 或其他任何字符串作为第二个参数输入，它都将工作。
