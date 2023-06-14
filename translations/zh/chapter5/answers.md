# Chapter 5 Questions and Answers

#### 1. 你认为 `select to_datetime(3600);` 将返回什么？为什么？

可以接受单个整数的函数签名是下面这个：

```
std::to_datetime(epochseconds: int64) -> datetime
```

由于它给出了 Unix Epoch (1970) 之后的秒数，它将返回：

`{<datetime>'1970-01-01T01:00:00Z'}`

即在纪元开始后一小时（3600 秒）。

#### 2. `select <int16>9 + 1.06n is decimal;` 能工作吗？如果可以，它将返回 `{true}` 吗？

可以工作，会返回 `{true}`。 EdgeDB 将选择 `decimal` 作为两种类型中更精确的。你也可以只用 `select <int16>9 + 1.06n;` 来查看类型，因为返回结果中有“n”：`{10.06n}`。

#### 3. 从 2003 年土库曼斯坦 (TMT) 圣诞节的早上 5:00 到乌兹别克斯坦 (UZT) 同年新年除夕夜的晚上 7:00 之间过去了多少秒？

使用 `to_datetime()` 函数很容易做到：

```edgeql
select to_datetime(2003, 12, 31, 19, 0, 0, 'UZT') - to_datetime(2003, 12, 25, 5, 0, 0, 'TMT');
```

答案是 158 小时：`{<duration>'158:00:00'}`。

因此，我们还可以将其乘以 3600 秒作为一个单独的查询，如：

```edgeql
select 158 * 3600;
```

我们将得到：`{568800}`。

或者我们还可以使用另一个名为 `duration_to_seconds` 的内置函数：

```edgeql
select duration_to_seconds(to_datetime(2003, 12, 31, 19, 0, 0, 'UZT') - to_datetime(2003, 12, 25, 5, 0, 0, 'TMT'));
```

我们将得到：`{568800.000000n}`。它仍然是 568,000 秒，但使用了更高的精度。

#### 4. 如何用对上题中的两个时间使用 `with` 写出同样查询效果的查询语句？

可以这样写 (取决于你给变量所起的名字及你倾向的顺序)：

```edgeql
with
  uzbek_time := (select to_datetime(2003, 12, 31, 19, 0, 0, 'UZT')),
  turkmen_time := (select to_datetime(2003, 12, 25, 5, 0, 0, 'TMT')),
select duration_to_seconds(uzbek_time - turkmen_time);
```

输出完全相同：`{568800.000000n}`。

#### 5. 如果您只想查看如何编写的某个类型，那么描述该类型的最佳方式是什么？

最佳方式是：`describe type as sdl`，没有 `as text` 给出的所有额外信息。以 `MinorVampire` 为例:

```
{
  'type default::MinorVampire extending default::Person {
    required link master: default::Vampire;
};',
}
```

它不显示来自它所扩展的 `Person` 类型的任何信息。
