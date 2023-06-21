---
tags: Indexing, String Functions
---

# 第十六章 - 伦菲尔德讲的是真的吗？

> 亚瑟·霍姆伍德（Arthur Holmwood）的父亲去世了，现在亚瑟成为了一家之主。他的新头衔是戈达尔明勋爵（Lord Godalming），他有很多很多钱。有了这笔钱，他可以帮助大家找出所有德古拉（Dracula）用来藏匿棺材的房子。

> 与此同时，范海辛（Van Helsing）对伦菲尔德（Renfield）感到好奇，于是他问约翰·西沃德（John Seward）是否可以安排见面。相见后，他很惊讶地发现伦菲尔德受到过良好的教育，而且口才很好。伦菲尔德谈论着范海辛的研究、政治、历史等——他看起来一点都不疯狂！但后来，伦菲尔德又突然不想说话了，不停地骂他是个白痴。这令范海辛感到很困惑。一天晚上，伦菲尔德非常认真地要求离开。他说：“你们难道不知道我是理智认真的吗？……是一个为灵魂而战的正常人！听我的，听我的，让我走！让我走！”。他们想相信他，但又无法说服自己真的信任。最后，伦菲尔德停了下来并平静地对大家说：“记住，今晚我已经尽力说服你们了。”

## 使用 index on 加快查询

我们越来越接近小说的尾声了，但仍有很多数据尚未得到输入。书中有很多有用的数据，只是我们还没想好如何整理。幸运的是，原版小说《德古拉》是以书信、日记等形式组织的，均以日期或时间开头。它们都是以下面这种方式开始的：

```
Dr. Seward’s Diary.
1 October, 4 a. m.—Just as we were about to leave the house...

Letter, Van Helsing to Mrs. Harker.
“24 September.
“Dear Madam...

Mina Murray’s Journal.
8 August. — Lucy was very restless all night, and I, too, could not sleep...
```

这就非常方便了。有了这个，我们可以创建一个类型来保存日期和书信中的字符串，以便我们之后进行搜索。我们称之为 `BookExcerpt`（excerpt = 摘录 = 一本书的一部分）。

```sdl
type BookExcerpt {
  required date: cal::local_datetime;
  required excerpt: str;
  index on (.date);
  required author: Person;
}
```

{ref}` ``index on (.date)`` <docs:ref_datamodel_indexes>` 部分是我们之前所没见过的，它意味着创建一个索引使之后的查询更加快速。使用 `index on` 的查找更快是因为数据库不再需要按顺序依次扫描整个对象集来找到匹配的对象。与总是扫描所有内容相比，索引可以更快地通过精确匹配进行查找（如果难以理解，可以想想字典中“A、B、C..."的作用）。

我们也可以给某些其他类型添加索引——它可能适用于诸如 `Place` 和 `Person` 之类的类型。

注意：`index` 在数量有限的情况下表现是良好的，你也不会希望索引所有的东西，因为：

- 虽然查询更快了，但增加了数据库大小；
- 如果你有太多索引，也会使 `insert` 和 `update` 变得很慢。

这并不奇怪，因为哪些需要使用 `index` 是用户需要做出的选择。如果在任何情况下使用 `index` 都是最好的主意，那么 EdgeDB 就自动执行了，不烦用户来做选择了。

最后，这里有两个不需要创建 `index` 的情况：

- 对于链接；
- 对于有排他性约束的属性。

在这两种情况下索引都会被自动创建，你无需为它们显式创建索引。

现在，让我们来插入两条摘录。这些条目中的字符串都很长（有时会长达几页），因此我们在这里仅显示开头和结尾：

```edgeql
insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 4, 0, 0),
  author := assert_single((select Person filter .name = 'John Seward')),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};
```

```edgeql
insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 5, 0, 0),
  author := assert_single((select Person filter .name = 'Jonathan Harker')),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```

然后我们可以执行下方查询，即按时间顺序获取所有条目并显示为 JSON。

```edgeql
select <json>(
  select BookExcerpt {
    date,
    author: {
      name
    },
    excerpt
  } order by .date
);
```

下面是仅包含一小部分摘录的 JSON 输出：

```
{
  "{\"date\": \"1893-10-01T04:00:00\", \"author\": {\"name\": \"John Seward\"}, \"excerpt\": \"Dr. Seward's Diary.\\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once...\\\"You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night.\\\"\"}",
  "{\"date\": \"1893-10-01T05:00:00\", \"author\": {\"name\": \"Jonathan Harker\"}, \"excerpt\": \"1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.\"}",
}
```

然后，我们可以在 `Event` 类型中添加一个新的链接，并指向我们的新类型——`BookExcerpt`。于是，`Event` 更新为：

```sdl
type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  multi excerpt: BookExcerpt; # Only this is new
  location: tuple<float64, float64>;
  east: bool;
  property url := get_url() ++ <str>.location.0 ++ '_N_' ++ <str>.location.1 ++ '_' ++ ('E' if .east else 'W');
}
```

这里的 `description` 用于记录我们编写的一个简短的字符串描述，而 `excerpt` 则链接到直接来自原著的较长的文本。

## 更多字符串函数

{ref}`字符串函数 <docs:ref_std_string>` 在对我们的 `BookExcerpt` 类型（或通过 `Event` 的 `BookExcerpt`）进行查询时特别有用。其中一个实用的函数是 {eql:func}`docs:std::str_lower`，它可以使字符串的字母都变为小写：

```
db> select str_lower('RENFIELD WAS HERE');
{'renfield was here'}
```

下面是一个较长的查询：

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(select len(.excerpt)) ++ ' characters'),
  the_date := (select (<str>.date)[0:10]),
} filter contains(str_lower(.excerpt), 'mina');
```

- 对 `.excerpt` 使用 `len()`，得到 `.excerpt` 所含字符的数量后，将其转换为字符串类型并赋予 `length`；
- 将 `cal::local_datetime` 类型的 `.date` 转换为字符串，再使用切片获取索引 0 到 9 的字符并赋予 `the_date`；
- 用 `str_lower()` 将 `.excerpt` 全部转换为小写，并使用 `contains()` 判断其是否包含 `mina` 来作为查询的过滤条件。

输出结果为：

```
{
  default::BookExcerpt {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1893-10-01',
  },
}
```

制作 `the_date` 的另一种方法是使用 {eql:func}`docs:std::to_str` 函数，你可能已经猜到了，它可以将 `cal::local_datetime` 类型的数据转换为字符串输出，且额外支持一个可选的参数输入，用于说明对输出字符串期望的输出格式：

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(select len(.excerpt)) ++ ' characters'),
  the_date := (select to_str(.date, 'YYYY-MM-DD')), # Only this part is different, and you don't have to pass the second parameter.
} filter contains(str_lower(.excerpt), 'mina');
```

可以作用于字符串的函数还有：

- `find()`：用于查找第二个字符串参数在第一个字符串参数中首次出现的位置索引，如果找不到任何匹配内容，则返回 `-1`：

`select find(BookExcerpt.excerpt, 'sofa');` 输出 `{-1, 151}`。这是因为我们有两个 `BookExcerpt` 对象，第一个 `BookExcerpt.excerpt` 中没有单词 `sofa`，而在第二个中的索引 151 处出现了 `sofa`。

- `str_split()`：该函数可以将字符串以你想要的方式进行拆分并转变为数组。最常见的是用 `' '` 作为分割符来分隔单词：

```
db> select str_split('Oh, hear me! hear me! Let me go! let me go! let me go!', ' ');
{
  [
    'Oh,',
    'hear',
    'me!',
    'hear',
    'me!',
    'Let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
  ],
}
```

下面的例子也同样工作（以 `'n'` 进行分割）：

```edgeql
select MinorVampire {
  names := (select str_split(.name, 'n'))
};
```

注意：结果中 `n` 都不见了：

```
{
  default::MinorVampire {names: ['Woma', ' 1']},
  default::MinorVampire {names: ['Woma', ' 2']},
  default::MinorVampire {names: ['Woma', ' 3']},
  default::MinorVampire {names: ['Lucy']},
}
```

你也可以以 `\n` 作为分割符按“新行”进行拆分。虽然你看不到它（`\n`），但从计算机的角度来看，每行的结尾都有一个 `\n`。例如：

```edgeql
select str_split('Oh, hear me!
hear me!
Let me go!
let me go!
let me go!', '\n');
```

`str_split` 将按“行”拆分第一个参数中的字符串并输出以下结果数组：

```
{['Oh, hear me!', 'hear me!', 'Let me go!', 'let me go!', 'let me go!']}
```

- `re_match()`（用于第一次匹配）和 `re_match_all()]`（用于所有匹配）：如果你对如何使用 [正则表达式](https://en.wikipedia.org/wiki/Regular_expression) 有所了解，并想使用它们，这俩函数可能会很有用。小说《德古拉》写于 100 多年前，因此有些单词的拼写已经不大一样了。例如，`tonight` 这个词在《德古拉》中总是使用较旧的 `to-night` 拼写。为了处理这样的问题，我们就可以使用这两个函数：

```
db> select re_match_all('[Tt]o-?night', 'Dracula is an old book, so the word tonight is written to-night. Tonight we know how to write both tonight and to-night.');
{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}
```

其中所用函数的签名是 `std::re_match_all(pattern: str, string: str) -> set of array<str>`，正如你所看到的那样，第一个参数是模式（pattern），第二个参数是被查找的字符串。模式 `[Tt]o-?night` 的意思是：

- 以 `T` 或 `t` 开头，
- 然后紧跟着一个 `o`，
- 也许 `o` 后面有一个 `-`，
- 并以 `night` 结束，

所以结果是：`{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}`。

如果你想匹配任何内容，可以使用通配符：`.`。

## 更多关于 `index on`

顺便说一下，`index on` 也可以用在你自己制作的表达式上。例如，如果我们总是需要查询一个 `City` 的名称及其人口，我们可以通过下面这种方式进行索引：

```sdl
type City extending Place {
  annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
  population: int64;
  index on (.name ++ ': ' ++ <str>.population);
}
```

另外，不要忘记你也可以为此添加注释。如果你担心代码的读者们可能不知道 `(.name ++ ': ' + <str>.population)` 的用途，则可以考虑为其添加注释：

```
type City extending Place {
    annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
    population: int64;
    index on (.name ++ ': ' ++ <str>.population) {
      annotation title := 'Lists city name and population for use in game function get_city_names';
    }
}
```

注释中提到的 `get_city_names`，我们并没有真正创建过；这里只是假设它在游戏中的某个地方被创建并使用，且重要到我们应该记住它。

[→ 点击这里查看到第 16 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何找出名称是由两个单词构成的所有 `Person` 对象，并将其名称拆分成两个字符串？

2. 如何找出所有名称里有“ma”的 `Person` 对象，并显示出其名称？

   提示：使用函数 `find()`。

3. 如何对 `Person` 类型的 `pen_name` 属性进行索引？

   提示：尝试使用 `describe type Person as SDL` 先查看一下它的 `pen_name` 属性。

4. 如何在展示所有 `Person` 对象的名称时，先以大写形式显示该名称，然后跟一个空格并以小写形式再显示一次其名称？

   提示：{eql:func}`docs:std::str_upper` 函数可以提供帮助。

5. 如何使用 `re_match_all()` 来显示名称中带有 `Crewman` 的所有 `Person.name`？例如：Crewman 1，Crewman 2，等等。

   提示：如果你想快速了解正则表达式，这里有一些 [基本概念](https://en.wikipedia.org/w/index.php?title=Regular_expression&oldid=988356211#Basic_concepts) 供你阅读。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _关于伦菲尔德的真相。_
