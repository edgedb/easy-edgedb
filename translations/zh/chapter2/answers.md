# Chapter 2 Questions and Answers

#### 1. 使用类型转换修改语句 `SELECT '99' + '1'`，使其输出结果为 `{100}`；

你可以用数字类型进行类型转换，比如 `SELECT <int64>'99' + <int64>'1'`。注意：这里的类型不是必须一样的，但是如果你使用了不同的数字类型，EdgeDB 将会自动对它们进行转换。你可以通过类似 `SELECT <int64>'99' + <int16>'1' IS int64;` 的查询来验证最终类型，这里它会返回 `{true}`：即如果你将它们加在一起，EdgeDB 会将较小的 `int16` 自动转换为较大的 `int64`。

#### 2. 选择出所有以“Mu”开头的 `City`（需要区分大小写）；

`SELECT City FILTER .name LIKE 'Mu%';`——别忘了 `%`，这样才可能匹配上类似名为 **Mu**nich 的城市。

#### 3. 选择出所有 `NPC` 名字的第三个字母（即索引号为 2）；

`SELECT NPC.name[2];`

#### 4. 你将如何修改 `Person` 类型成为 `HasAString` 的扩展类型？

添加 `extending HasAString` 到 `Person` 的定义里，如下所示：

```sdl
abstract type Person extending HasAString {
  required property name -> str;
  multi link places_visited -> City;
}
```

#### 5. 下面的查询仅会显示造访过的地方的 id。请问如何显示它们的名字？

修改为：

```edgeql
SELECT Person {
  places_visited: {name}
};
```

别忘了 `:`，且空格间距无关紧要，因此你也可以写为：

```edgeql
SELECT Person {
  places_visited: {
    name
    }
};
```
