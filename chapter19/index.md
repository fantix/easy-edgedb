---
tags: Reverse Links, Schema Cleanup
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
> “那你在船上？”
>
> “哦，是的！”

> 现在他们知道了德古拉带着他的最后一个棺材箱子逃到了一艘船上，准备返回特兰西瓦尼亚（Transylvania）。范海辛和米娜前往德古拉城堡（Castle Dracula），而其他人则前往瓦尔纳（Varna）试图在船到达时抓住德古拉。乔纳森·哈克（Jonathan Harker）磨好了刀，他现在看起来完全变了个人。但是船在哪里？他们每天都在等待……然后有一天，他们收到一条消息：船到达了河上游的加拉茨（Galatz），而不是瓦尔纳！他们来晚了吗？他们冲到河的上游去寻找德古拉。

## 添加一些新类型（Adding some new types）

在本章中，出现了另一艘船在地图上移动。之前，我们制作了一个 `Ship` 类型，如下所示：

```sdl
type Ship extending HasNameAndCoffins {
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

这还不错，但我们可以在本章中进一步构建它。目前，我们还没有任何关于哪艘船到过哪里的信息。让我们创建一个包含所有船舶访问信息的快速类型。每次访问都会有一个指向 `Ship` 的链接和一个指向 `Place` 的链接，以及一个 `cal::local_date`。如下所示：

```sdl
type Visit {
  required link ship -> Ship;
  required link place -> Place;
  required property date -> cal::local_date;
}
```

德古拉乘坐的这艘新船被称为 `Czarina Catherine`（沙皇（Czarina）是俄罗斯女王）。让我们用上面的类型来插入我们所知道的船只的一些访问记录。你应该还记的另一艘叫做德米特号的船，它从瓦尔纳开往伦敦。

但首先我们先插入新的 `Ship` 和两个新的地点（`City`），以便我们可以链接它们。我们知道这艘船的名字，且上面有一口棺材：德古拉的最后一口棺材。但是我们不知道船员的情况，所以我们只插入以下信息：

```edgeql
INSERT Ship {
  name := 'Czarina Catherine',
  coffins := 1,
};
```

之后我们需要创建瓦尔纳（Varna）和加拉茨（Galatz）两个城市。我们一次性将他们插入：

```edgeql
FOR city in {'Varna', 'Galatz'}
UNION (
  INSERT City {
    name := city
  }
);
```

德米特号（the Demeter）船长的书中还提及了很多其他的地方，我们来看看其中的几个。德米特号曾穿过博斯普鲁斯海峡（Bosphorus）。该地区是土耳其将欧洲与亚洲分开的一小块海洋，因此它不是一个城市。我们可以使用 `OtherPlace` 类型，这也是我们在第 14 章中添加了一些注解的类型。还记得如何调用它们吗？如下所示：

```edgeql
SELECT (INTROSPECT OtherPlace) {
  name,
  annotations: {
    @value
  }
};
```

让我们看看输出，看看我们之前写了什么，以确保我们能够使用它：

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

嗯，它不是一座城堡，它实际上也不是一个有建筑物的地方，所以使用它很合适。很快我们将会创建一个 `Region` 类型，以便我们可以有一个 `Country` -> `Region` -> `City` 布局。在这种情况下，`OtherPlace` 可能会在 `Country` 或 `Region` 中被链接。但在此，我们将只添加它而不链接到任何其他：

```edgeql
INSERT OtherPlace {
  name := 'Bosphorus'
};
```

很简单。现在我们可以输入船舶的访问记录。

```edgeql
FOR visit in {
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

有了这些数据，现在我们的游戏可以在特定日期各城市的船只停泊情况。例如，假设一个角色进入了加拉茨（Galatz）。如果日期是 1887 年 10 月 28 日，我们可以查看镇上是否有船只：

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

看起来镇上确实有一艘船！正是沙皇凯瑟琳。

```
{
  Object {
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

`Ship.<ship[IS Visit]` 是指所有带有指向 `Ship` 类型的链接 `ship` 的 `Visits`。因为我们选择的是 `Visit` 而不是 `Ship`，所以我们的过滤器现在是作用在 `Visit` 的 `.place.name` 上，而不是 `Ship` 中的属性。

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

顺便说一下，故事的主人公们通过瓦尔纳市一家公司发来的电报得知了 `Czarina Catherine` 的情况。电报中说：

```
28 October.—Telegram. Rufus Smith, London, to Lord Godalming, care H. B. M. Vice Consul, Varna.

“Czarina Catherine reported entering Galatz at one o’clock to-day.”
```

还记得我们的 `Time` 类型吗？我们制作它是为了我们主需要输入一个字符串即可获得一些有用的信息。现在看来，它几乎是一个函数：

```sdl
type Time {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

现在我们知道时间是下午 1 点钟，让我们将其也放入查询中 - 包括 `awake` 属性。如下所示：

```edgeql
SELECT Ship.<ship[IS Visit] {
  place: {
    name
  },
  ship: {
    name
  },
  date,
  time := (
    SELECT (
      Insert Time {
        date := '13:00:00'
      }
    ) {
      date,
      local_time,
      hour,
      awake
    }
  ),
} FILTER .place.name = 'Galatz';
```

下面是输出，包括吸血鬼是醒着还是睡着了。

```
{
  default::Visit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1887-10-28',
    time: default::Date {date: '13:00:00', local_time: <cal::local_time>'13:00:00', hour: '13', awake: 'asleep'},
  },
}
```

## 更多架构清理（More cleaning up the schema）

像上面那样在查询中快速做插入当然很酷，但这有点奇怪。问题是我们现在有的是一个随机的 `Time` 对象，它没有链接到任何东西。为此，让我们把 `Time` 中的所有属性拿出来用于改进 `Visit` 类型。


```sdl
type Visit {
  link ship -> Ship;
  link place -> Place;
  required property date -> cal::local_date;
  property time -> str;
  property local_time := <cal::local_time>.time;
  property hour := .time[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

然后更新对加拉茨（Galatz）的访问，给它一个 `time`：

```edgeql
UPDATE Visit FILTER .place.name = 'Galatz'
SET {
  time := '13:00:00'
};
```

然后我们将再次使用反向查询。我们再添加一个可计算的公式（给到 `when_arthur_got_the_telegram`）以增加一些趣味性：假设亚瑟（Arthur）花了两小时五分十秒才收到电报。我们将 `time` 字符串转换为 `cal::local_time`，然后添加一个 `duration`。

```edgeql
SELECT Ship.<ship[IS Visit] {
  place: {
    name
  },
  ship: {
    name
  },
  date,
  time,
  hour,
  awake,
  when_arthur_got_the_telegram := (<cal::local_time>.time) + <duration>'2 hours, 5 minutes, 10 seconds'
} FILTER .place.name = 'Galatz';
```

现在我们得到了之前 `Time` 类型给我们的所有输出，以及我们关于亚瑟（Arthur）何时收到电报的额外信息：

```
{
  default::Visit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1887-10-28',
    time: '13:00:00',
    hour: '13',
    awake: 'asleep',
    when_arthur_got_the_telegram: <cal::local_time>'15:05:10',
  },
}
```

我们再来看一下 `Place`，现在我们可以通过使用我们讨论过的 `Region` 类型填写地图来完成本章。这很容易：

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

这很好地连接了我们基于 `Place` 的各类型。

现在让我们做一个同时包含 `Country`、`Region` 和 `City` 的中等条目。我们将选择 1887 年的德国，因为乔纳森首先经过了那里。它将有：

- 1 个国家：德国（Germany），
- 3 个地区：普鲁士（Prussia）、黑森（Hesse）、萨克森（Saxony），
- 每个地区有两个城市，共计 6 个城市：柏林（Berlin）和柯尼斯堡（Königsberg）、达姆施塔特（Darmstadt）和美因茨（Mainz）、德累斯顿（Dresden）和莱比锡（Leipzig）。


Here is the insert:

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

有了这个漂亮的结构的搭建，我们可以做一些事情，比如选择一个 `Region` 并查看其中的城市，以及它所属的国家。要从 `Region` 获取 `Country`，我们需要反向查找：

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

通过最后的反向查找，我们在 `Country` 和它的属性 `regions` 之间有了另一个链接，现在我们得到了 `Country` 作为输出。这是输出：

```
{
  Object {
    name: 'Prussia',
    cities: {default::City {name: 'Berlin'}, default::City {name: 'Königsberg'}},
    country: {default::Country {name: 'Germany'}},
  },
  Object {
    name: 'Hesse',
    cities: {default::City {name: 'Darmstadt'}, default::City {name: 'Mainz'}},
    country: {default::Country {name: 'Germany'}},
  },
  Object {
    name: 'Saxony',
    cities: {default::City {name: 'Dresden'}, default::City {name: 'Leipzig'}},
    country: {default::Country {name: 'Germany'}},
  },
}
```

[点击这里查看第 19 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何显示所有 `City` 名称和它们所在的 `Region` 名称？

2. 基于上一题，如何显示所有 `City` 名称和它们所在的 `Region` 名称及 `Country` 名称？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _与时间赛跑。_
