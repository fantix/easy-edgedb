---
tags: Defaults, Overloading, For Loops
---

# Chapter 9 - 在英格兰发生的奇怪事

在本章中，我们回到了几周前，船刚刚离开瓦尔纳（Varna）而米娜（Mina）和露西（Lucy）还没有启程前往去惠特比（Whitby）的时候。故事情节也分为两部分介绍。这是第一个：

> 我们仍然不知道乔纳森（Jonathan）在哪里，德米特号（The Demeter）船正在前往英格兰（England）的途中，德古拉（Dracula）也在船上。与此同时，米娜（Mina）正在伦敦给她的朋友 露西·韦斯特拉（Lucy Westenra）写信。露西有三个男朋友，分别是约翰·苏厄德医生（Dr. John Seward）、昆西·莫里斯（Quincey Morris）和亚瑟·霍姆伍德（Arthur Holmwood），他们都想娶她……

## 关于日期的更多处理（Working with dates some more）

按照故事情节的发展，看起来我们还有更多人物需要插入。但首先，让我们再思考一下那艘船。船上所有人都被德古拉（Dracula）杀死了，但我们并不想删除船员，因为他们仍然是我们游戏的一部分。小说告诉我们，这艘船是在 7 月 6 日离开的，最后一个人（船长）死于 8 月 4 日（1887 年）。

这正是给 `Person` 类型添加两个新属性的好时机，以展示一个角色存在的时间。我们给它们命名为 `first_appearance` 和 `last_appearance`。`last_appearance` 比起 `death` 更为合适，因为对于游戏来说这无关紧要：我们只想知道角色何时在场。

对于这两个属性，为了简单起见，我们将只使用 `cal::local_date`。还有包含了时间的 `cal::local_datetime` 类型，但我们应该只用得到日期。（当然还有 `cal::local_time` 类型，它只是我们在 `Date` 类型中拥有的一天中的时间。）

对具有属性 `first_appearance` 和 `last_appearance` 的 `Crewman` 对象进行插入，如下所示：

```edgeql
INSERT Crewman {
  number := count(DETACHED Crewman) +1,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
};
```

由于我们已经插入了很多 `Crewman` 对象，如果我们假设他们都同时死亡（或者并不需要那么精确），我们可以轻松地对所有这些对象使用 `UPDATE` 和 `SET`。

由于 `cal::local_date` 具有非常简单的 YYYYMMDD 格式，在插入中使用它的最简单方法就是从字符串进行转换：

```edgeql
SELECT <cal::local_date>'1887-07-08';
```

但是我们之前使用过一个函数，可以将单独的数字输入到函数中，因此我们将继续使用该方法。

之前我们使用的是带有七个参数的函数 `std::to_datetime` ；这次我们将使用类似但更短的 [`cal::to_local_date`](https://www.edgedb.com/docs/edgeql/funcops/datetime#function::cal::to_local_date) 函数。它只需要三个整数。

这是它的签名（我们在使用第三个）：

```
cal::to_local_date(s: str, fmt: OPTIONAL str = {}) -> local_date
cal::to_local_date(dt: datetime, zone: str) -> local_date
cal::to_local_date(year: int64, month: int64, day: int64) -> local_date
```

现在我们更新 `Crewman` 对象并给它们相同的日期以保持简单：

```edgeql
UPDATE Crewman
SET {
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16)
};
```

当然这些日期取决于我们的游戏。一个 `PC` 实际上可以在这艘船航行到英格兰时登船访问它吗？在德古拉杀死船员之前，会有试图拯救船员的任务吗？如果是这样，那么我们将需要更精确的日期。但现在，这些大致日期足够了。

## 为类型添加默认值以及重载关键字（Adding defaults to a type, and the overloaded keyword）

现在让我们回到对新角色的插入。首先，我们将插入露西（Lucy）：

```edgeql
INSERT NPC {
  name := 'Lucy Westenra',
  places_visited := (SELECT City FILTER .name = 'London')
};
```

嗯，看起来每当我们添加一个角色，我们都要做很多工作来插入 'London'。我们还剩下三个角色，他们也都来自伦敦。为了节省一些工作，我们可以将伦敦设为 `NPC` 的 `places_visited` 的默认值。为此，我们需要两件事：用 `default` 声明默认值，以及使用关键字 `overloaded`。`overloaded` 这个词表明我们使用 `placed_visited` 的方式不同于我们扩展自的 `Person` 类型。

添加了 `default` 和 `overloaded` 后，看起来像这样：

```sdl
type NPC extending Person {
  property age -> HumanAge;
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

## datetime_current()

这有一个方便的函数是 [datetime_current()](https://www.edgedb.com/docs/edgeql/funcops/datetime/#function::std::datetime_current)，它可以给出了现在的日期时间。让我们试试看：

```edgeql-repl
edgedb> SELECT datetime_current();
{<datetime>'2020-11-17T06:13:24.418765000Z'}
```

如果在插入对象时你需要一个发布日期，这会很有用。有了这个，你可以按日期排序，如果有重复项，则删除最近插入的条目，等等。让我们想象一下如果我们把它放在 `Place` 类型中会是什么样子。如下，很接近，但不完全是：

```sdl
abstract type Place {
  required property name -> str {
    constraint exclusive;
  }
  property modern_name -> str;
  property important_places -> array<str>;
  property post_date := datetime_current(); # this is new
}
```

这实际上会在你*查询* `Place` 对象时生成日期，而不是在你插入它时。因此，要创建一个带有插入日期的 `Place` 类型，我们可以使用 `default` 代替：

```sdl
abstract type Place {
  required property name -> str {
    constraint exclusive;
  }
  property modern_name -> str;
  property important_places -> array<str>;
  property post_date -> datetime {
    default := datetime_current()
  }
}
```

在我们的架构（schema）中我们并不需要这个日期，所以我们并不去真的改变 `Place`，这里只是为了展示你可以如何操作。

## 使用 FOR 和 UNION（Using FOR and UNION）

我们几乎准备好插入我们的新角色了，现在我们不需要每次都添加`(SELECT City FILTER .name = 'London')`。但是，如果我们可以使用单个插入而不是三个插入不是很好吗？

要做到这一点，我们可以使用 `FOR` 循环，后跟关键字 `UNION`。首先，这是 `FOR` 部分：
To do this, we can use a `FOR` loop, followed by the keyword `UNION`. First, here's the `FOR` part:

```edgeql
FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
```

换句话说：获取这三个字符串组成的集合，并对每个字符串做一些事情。`character_name` 是我们选择调用这个集合中的每个字符串所用的变量名称。

`UNION` 紧随其后，因为它是用于将集合连接在一起的关键字。例如，这个查询：

```edgeql
WITH city_names := (SELECT City.name),
  castle_names := (SELECT Castle.name),
SELECT city_names UNION castle_names;
```

将名称集合合并在一起，从而输出：`{'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Castle Dracula'}`。

现在让我们回到带有变量名 `character_name` 的 `FOR` 循环，它看起来像这样：

```edgeql
FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
UNION (
  INSERT NPC {
    name := character_name,
    lover := (SELECT Person FILTER .name = 'Lucy Westenra'),
  }
);
```

我们会得到三个 `uuid` 作为响应，表示三个角色已被输入。

然后让我们检查一下以确保它确实成功了：

```edgeql
SELECT NPC {
  name,
  places_visited: {
    name,
  },
  lover: {
    name,
  },
} FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'};
```

正如我们所希望的那样，他们现在都与露西有关。

```
{
  Object {
    name: 'John Seward',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
  Object {
    name: 'Quincey Morris',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
  Object {
    name: 'Arthur Holmwood',
    places_visited: {Object {name: 'London'}},
    lover: Object {name: 'Lucy Westenra'},
  },
}
```

顺便说一下，现在我们可以使用这个方法将我们的五个 `Crewman` 对象用一个 `INSERT` 完成插入，而不是 `INSERT` 五次。我们可以将船员的编号放在一个集合中，并使用 `FOR` 和 `UNION` 来插入他们。当然，在之前我们已经使用过 `UPDATE` 更改了插入，但从现在开始，在我们的代码中，船员的插入将如下所示：

```edgeql
FOR n IN {1, 2, 3, 4, 5}
UNION (
  INSERT Crewman {
    number := n
    first_appearance := cal::to_local_date(1887, 7, 6),
    last_appearance := cal::to_local_date(1887, 7, 16),
  }
);
```

使用 `FOR` 时，最好熟悉 [要遵循的顺序](https://www.edgedb.com/docs/edgeql/statements/for#for)：

```edgeql-synopsis
[ WITH with-item [, ...] ]

FOR variable IN "{" iterator-set [, ...]  "}"

UNION output-expr ;
```

重要的部分是 `{` 和 `}`，因为 `FOR` 只用于集合。如果你尝试使用数组或其他类型，则会出现错误。

现在是时候对露西（Lucy）更新三个情人了。露西已经破坏了我们将 `lover` 仅仅作为一个 `link`（这意味着 `single link`）的设定。我们要将其设置为 `multi link`，这样我们就可以添加所有三个人了。这里是我们对她的更新：

```edgeql
UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (
    SELECT Person FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};
```

现在我们查询她以验证更新有效。这次让我们在做过滤器的时候使用 `LIKE`：

```edgeql
SELECT NPC {
  name,
  lover: {
    name
  }
} FILTER .name LIKE 'Lucy%';
```

这确实把她和她的三个情人打印出来了。

```
{
  Object {
    name: 'Lucy Westenra',
    lover: {
      Object {name: 'John Seward'},
      Object {name: 'Quincey Morris'},
      Object {name: 'Arthur Holmwood'},
    },
  },
}
```

## 用重载替代新类型的创建（Overloading instead of making a new type）

所以现在我们知道关键字 `overloaded`，我们不再需要 `NPC` 中用到的 `HumanAge` 类型了。`HumanAge` 长这样：

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

你应该还记得我们制作这个类型是因为吸血鬼可以永生，但人类只能活到 120 岁。但现在我们对其进行简化。首先，我们将 `age` 属性移到 `Person` 类型。然后（在 `NPC` 类型内）我们使用 `overloaded` 对 `age` 添加一个约束。现在 `NPC` 里使用了两个 `overloaded`：

```sdl
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City filter .name = 'London');
  }
}
```

这很方便，因为我们也可以从 `Vampire` 中删除 `age` 了：

```sdl
type Vampire extending Person {
  # property age -> int16; **Delete this one now**
  multi link slaves -> MinorVampire;
}
```

你可以看到，如果你可以正确使用抽象类型和关键字 `overloaded`，你的架构可以被简化。

好的，接下来让我们阅读本章介绍的剩余部分。它继续解释了露西（Lucy）在做什么：

> ……她选择嫁给亚瑟·霍姆伍德（Arthur Holmwood），并向另外两人道歉。另外两个男人很难过，好在他们成为了彼此的朋友。苏厄德医生（Dr. Seward）很沮丧，并试图专注于他的工作以摆脱情伤。他是一名精神病医生，在伦敦郊外不远处的一座名为 Carfax 的大宅邸附近的精神病院工作。疯人院里有个奇怪的人，名叫伦菲尔德（Renfield），苏厄德医生觉得他最有趣。雷菲尔德有时冷静，有时癫狂，苏厄德医生不知道为什么他的情绪变化如此之快。此外，伦菲尔德似乎相信他可以通过吃活物来获得力量。他不是吸血鬼，但有时看起来很相似。

哎呀！看起来露西（Lucy）已经没有三个情人了。现在我们必须将她更新为只有亚瑟（Arthur）一个情人：

```edgeql
UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (SELECT NPC FILTER .name = 'Arthur Holmwood'),
};
```

然后将她从另外两个“备胎”中移除。我们只好给他们一个悲伤的空集了。

```edgeql
UPDATE NPC FILTER .name in {'John Seward', 'Quincey Morris'}
SET {
  lover := {} # 😢
};
```

现在看起来基本上都是最新了。就剩下插入神秘的伦菲尔德（Renfield）了。这很容易，因为他没有情人，不需要做 `FILTER`：

```edgeql
INSERT NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1887, 5, 26),
  strength := 10,
};
```

但他与德古拉似乎有某种关系，类似于 `MinorVampire` 类型但又不同。他也很强壮（稍后我们会看到），所以我们给他的 `strength` 设置为 10。稍后我们将对他以及他与德古拉的关系有更进一步的了解。

[→ 点击这里查看第 9 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 为什么下面这个插入不起作用，该如何修复？

   ```edgeql
   FOR castle IN ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
   UNION (
     INSERT Castle {
       name := castle
     }
   );
   ```

2. 如何在显示城堡名称的同时进行与上题相同的插入？
3. 如果所有的吸血鬼都需要一个最小为 10 的力量值，如何修改 `Vampire` 类型？
4. 如何更新所有的 `Person` 类型的对象，表明他们都死于 1887 年 9 月 11 日？

   提示：这里是 `Person` 类型的定义：

   ```sdl
   abstract type Person {
     required property name -> str {
       constraint exclusive;
     }
     property age -> int16;
     property strength -> int16;
     multi link places_visited -> Place;
     multi link lover -> Person;
     property first_appearance -> cal::local_date;
     property last_appearance -> cal::local_date;
   }
   ```

5. 所有名字中带有 `e` 或 `a` 的 `Person` 角色都被复活了。对此你将如何更新？

   提示：“复活”意味着 `last_appearance` 应该返回 `{}`。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _大雾和风暴袭击了惠特比市。_
