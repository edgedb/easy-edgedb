---
tags: Backlinks, Schema Cleanup
---

# 第十九章 - 德古拉逃了

> 范海辛（Van Helsing）催眠了米娜（Mina），米娜现在是半个吸血鬼，她可以感知到德古拉（Dracula）。范海辛问她：

> “你现在在哪里？”
>
> “我不知道。这一切对我来说都很奇怪！”
>
> “你看到了什么？”
>
> “我什么也看不见；一片漆黑。”
>
> “你听到了什么？”
>
> “水声……还有小浪。”
>
> “你在船上？”
>
> “哦，是的！”

> 现在大家明白了，德古拉带着他的最后一个棺材逃到了一艘船上，准备返回特兰西瓦尼亚（Transylvania）。范海辛和米娜前往德古拉城堡（Castle Dracula），而其他人则前往瓦尔纳（Varna）试图在船到达时抓住德古拉。乔纳森·哈克（Jonathan Harker）磨好了刀，他现在看起来完全变了个人，一心只想杀了德古拉并拯救自己的妻子米娜。但是船到哪了？他们每天都在等待……这一天，他们收到一条消息称：船抵达了上游的加拉茨（Galatz），而不是瓦尔纳！他们来晚了吗？他们冲到河的上游去寻找德古拉。

## 添加一些新类型

在本章中，出现了另一艘船在地图上移动。之前，我们制作了一个 `Ship` 类型，如下所示：

```sdl
type Ship extending HasNameAndCoffins {
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

看起来还不错，但我们可以在本章中进一步构建它。目前，我们还没有任何关于什么船到过什么地方的信息。现在让我们来创建一个包含所有船舶访问信息的快速类型 `Visit`。每一次的访问都会有一个指向 `Ship` 的链接和一个指向 `Place` 的链接，以及一个用来展示访问日期的 `cal::local_date`。如下所示：

```sdl
type Visit {
  required link ship -> Ship;
  required link place -> Place;
  required property date -> cal::local_date;
}
```

德古拉乘坐的这艘新船被称为 `Czarina Catherine`（沙皇（Czarina）是俄罗斯女王）。现在，让我们用上面的类型来创建已知船只的访问记录。比如，你应该还记的另一艘叫做德米特号（the Demeter）的船，它由瓦尔纳开往伦敦。

这里，让我们先来创建一下新出现的 `Ship` 和两个新的地点 `City`，以便我们可以链接它们。我们知道这艘船的名字，且上面有一个棺材：德古拉的最后一具棺材。但是我们不知道船员的情况，所以我们只需插入以下信息：

```edgeql
INSERT Ship {
  name := 'Czarina Catherine',
  coffins := 1,
};
```

然后，我们需要创建瓦尔纳（Varna）和加拉茨（Galatz）两个城市：

```edgeql
FOR city IN {'Varna', 'Galatz'}
UNION (
  INSERT City {
    name := city
  }
);
```

德米特号（the Demeter）船长的日记中还提及了很多其他的地方，我们来看看都有哪些：德米特号曾穿过博斯普鲁斯海峡（Bosphorus）。该地区是土耳其将欧洲与亚洲分开的一小块海洋，因此它不是一个城市。我们可以使用 `OtherPlace` 类型，这也是我们在第 14 章中添加了一些注解的类型。还记得如何显示注解吗？如下所示：

```edgeql
SELECT (INTROSPECT OtherPlace) {
  name,
  annotations: {
    @value
  }
};
```

让我们从输出中看看我们之前写了什么，以确保我们能够正确使用它：

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {
        @value: 'A place with under 50 buildings - hamlets, small villages, etc.',
      },
      schema::Annotation {
        @value: 'Castles and castle towns do not count! Use the Castle type for that',
      },
    },
  },
}
```

嗯，博斯普鲁斯海峡（Bosphorus）不是一座城堡，也不是一个有建筑物的地方，所以使用 `OhterPlace` 很合适。后面我们将会创建一个 `Region` 类型，以便我们可以有一个 `Country` -> `Region` -> `City` 的布局。在这种情况下，`OtherPlace` 可能会在 `Country` 或 `Region` 中被链接。但目前，我们先只是添加它，暂不需要链接到任何其他类型：

```edgeql
INSERT OtherPlace {
  name := 'Bosphorus'
};
```

这很简单。现在我们可以输入船舶的访问记录了。

```edgeql
FOR visit IN {
    ('The Demeter', 'Varna', '1887-07-06'),
    ('The Demeter', 'Bosphorus', '1887-07-11'),
    ('The Demeter', 'Whitby', '1887-08-08'),
    ('Czarina Catherine', 'London', '1887-10-05'),
    ('Czarina Catherine', 'Galatz', '1887-10-28')
  }
UNION (
  INSERT Visit {
    ship := (SELECT Ship FILTER .name = visit.0),
    place := (SELECT Place FILTER .name = visit.1),
    date := <cal::local_date>visit.2
  }
);
```

有了这些数据，我们在游戏中可以随时查看特定日期下各城市的船只停泊情况了。例如，假设一个角色进入了加拉茨（Galatz）。如果日期是 1887 年 10 月 28 日，我们则可以通过下面的语句查看当地是否有船只停靠：

```edgeql
SELECT Visit {
  ship: {
    name
  },
  place: {
    name
  },
  date
} FILTER .place.name = 'Galatz' AND .date = <cal::local_date>'1887-10-28';
```

看起来镇上确实有一艘船！正是沙皇凯瑟琳（Czarina Catherine）。

```
{
  default::Visit {
    ship: default::Ship {name: 'Czarina Catherine'},
    place: default::City {name: 'Galatz'},
    date: <cal::local_date>'1887-10-28',
  },
}
```

现在，让我们在船只的访问中再次练习一下反向查找。比如：

```edgeql
SELECT Ship.<ship[IS Visit] {
  place: {
    name
  },
  ship: {
    name
  },
  date
} FILTER .place.name = 'Galatz';
```

`Ship.<ship[IS Visit]` 是指所有通过链接 `ship` 指向了某个 `Ship` 类型对象的 `Visit`。因为我们选择的是 `Visit` 而不是 `Ship`，所以我们的过滤器此时是作用在 `Visit` 的 `.place.name` 上，而不是 `Ship` 的属性上。

这是输出：

```
{
  default::Visit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1887-10-28',
  },
}
```

顺便说一下，瓦尔纳市内的一家公司给故事的主人公们发了一封电报并说明了 `Czarina Catherine` 的情况。电报中说：

```
28 October.—Telegram. Rufus Smith, London, to Lord Godalming, care H. B. M. Vice Consul, Varna.

“Czarina Catherine reported entering Galatz at one o’clock to-day.”
```

还记得我们的 `Time` 类型吗？我们制作它是为了我们可以输入一个字符串就获得一些有用的信息。现在看来，它几乎是一个函数：

```sdl
type Time {
  required property clock -> str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

电报里提到的抵达时间是下午 1 点钟，让我们将该信息也放入到查询中，并获取包括 `awake` 属性在内的时间信息。如下所示：

```edgeql
WITH time := (
  INSERT Time {
    clock := '13:00:00'
  }
)
SELECT Ship.<ship[IS Visit] {
  place: {
    name
  },
  ship: {
    name
  },
  date,
  time_ := time {
    clcok,
    clock_time,
    hour,
    awake
  },
} FILTER .place.name = 'Galatz';
```

输出如下所示，包括了吸血鬼是醒着还是睡着。

```
{
  default::Visit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1887-10-28',
    time_: default::Time {
      clock: '13:00:00',
      clock_time: <cal::local_time>'13:00:00',
      hour: '13',
      awake: 'asleep',
    },
  },
}
```

## 更多架构清理

像上面那样在查询中快速做插入当然很酷，但这有点奇怪。问题在于我们现在使用的是一个随机的 `Time` 对象，它没有链接到任何东西。为此，让我们把 `Time` 中的所有属性都拿出来用于改进 `Visit` 类型。

```sdl
type Visit {
  link ship -> Ship;
  link place -> Place;
  required property date -> cal::local_date;
  property clock -> str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

然后更新对加拉茨（Galatz）的访问，给它一个 `clock`：

```edgeql
UPDATE Visit FILTER .place.name = 'Galatz'
SET {
  clock := '13:00:00'
};
```

然后，我们将再次使用反向查询来完成同上面一样的对 `Visit` 的查询。同时，我们再添加一个计算（computed）属性给到 `when_arthur_got_the_telegram` 以增加一些趣味性：假设亚瑟（Arthur）花了两小时五分十秒才收到电报。我们将 `time` 字符串转换为 `cal::local_time`，然后再为其加上一个 `duration`，即可得到 `when_arthur_got_the_telegram`。

```edgeql
SELECT Ship.<ship[IS Visit] {
  place: {
    name
  },
  ship: {
    name
  },
  date,
  clock,
  hour,
  awake,
  when_arthur_got_the_telegram := (<cal::local_time>.clock) + <duration>'2 hours, 5 minutes, 10 seconds'
} FILTER .place.name = 'Galatz';
```

现在，我们如愿得到了之前 `Time` 类型可以给到我们的所有输出，以及我们关于亚瑟（Arthur）何时收到电报的额外信息：

```
{
  default::Visit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1887-10-28',
    clock: '13:00:00',
    hour: '13',
    awake: 'asleep',
    when_arthur_got_the_telegram: <cal::local_time>'15:05:10',
  },
}
```

我们再来看一下 `Place`，现在让我们来创建之前提到的 `Region` 类型以进一步填满我们的游戏地图。这很容易：

```sdl
type Country extending Place {
  multi link regions -> Region;
}

type Region extending Place {
  multi link cities -> City;
  multi link other_places -> OtherPlace;
  multi link castles -> Castle;
}
```

它将很好地连接我们基于 `Place` 扩展出的各个类型。

现在让我们来做一条同时包含 `Country`、`Region` 和 `City` 的数据。我们将选择 1887 年的德国，因为故事的最初乔纳森就经过了那里。它将有：

- 1 个国家：德国（Germany），
- 3 个地区：普鲁士（Prussia）、黑森（Hesse）、萨克森（Saxony），
- 每个地区有两个城市，共计 6 个城市：柏林（Berlin）和柯尼斯堡（Königsberg）、达姆施塔特（Darmstadt）和美因茨（Mainz）、德累斯顿（Dresden）和莱比锡（Leipzig）。

插入语句如下所示：

```edgeql
INSERT Country {
  name := 'Germany',
  regions := {
    (INSERT Region {
      name := 'Prussia',
      cities := {
        (INSERT City {
          name := 'Berlin'
        }),
        (INSERT City {
          name := 'Königsberg'
        }),
      }
    }),
    (INSERT Region {
      name := 'Hesse',
      cities := {
        (INSERT City {
          name := 'Darmstadt'
        }),
        (INSERT City {
          name := 'Mainz'
        }),
      }
    }),
    (INSERT Region {
      name := 'Saxony',
      cities := {
        (INSERT City {
          name := 'Dresden'
        }),
        (INSERT City {
          name := 'Leipzig'
        }),
      }
    })
  }
};
```

搭建了这个漂亮的结构后，我们可以做一些额外的事情，比如选择一个 `Region` 并查看其中的城市，以及它所属的国家。要从 `Region` 获取 `Country`，我们需要使用反向查找：

```edgeql
SELECT Region {
  name,
  cities: {
    name
  },
  country := .<regions[IS Country] {
    name
  }
};
```

通过反向查找，我们在 `Country` 和它的属性 `regions` 之间拥有了另一种链接，现在我们在查询 `Region` 是反向获取了其所属的 `Country` 信息作为输出，如下所示：

```
{
  default::Region {
    name: 'Prussia',
    cities: {default::City {name: 'Berlin'}, default::City {name: 'Königsberg'}},
    country: {default::Country {name: 'Germany'}},
  },
  default::Region {
    name: 'Hesse',
    cities: {default::City {name: 'Darmstadt'}, default::City {name: 'Mainz'}},
    country: {default::Country {name: 'Germany'}},
  },
  default::Region {
    name: 'Saxony',
    cities: {default::City {name: 'Dresden'}, default::City {name: 'Leipzig'}},
    country: {default::Country {name: 'Germany'}},
  },
}
```

[→ 点击这里查看到第 19 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何显示所有 `City` 的名称和它们所在 `Region` 的名称？

2. 基于上一题，如何显示所有 `City` 的名称和它们所在 `Region` 及 `Country` 的名称？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _与时间赛跑。_
