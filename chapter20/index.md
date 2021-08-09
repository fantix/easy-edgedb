---
tags: Ddl, Sdl, Edgedb Community
---

# 第二十章 - 最后一战

终于，我们来到了最后一章 —— 恭喜！下面是本章的最后一幕，但我们并不打算剧透最终的结局：

> 米娜（Mina）现在几乎是一个吸血鬼，她说无论一天中的什么时候，她都能感觉到德古拉（Dracula）。范海辛（Van Helsing）抵达德古拉城堡，米娜在外面等候。范海辛进到城堡摧毁了吸血鬼女人和德古拉的棺材。与此同时，其他人从南方赶来，也将抵达德古拉城堡。德古拉的朋友们把他放在他的盒子（棺材）里，并用一辆马车以最快的速度把他运回城堡。太阳快落山了，下起了雪，必须尽快抓到德古拉。他们越来越靠近，终于抓住了盒子。他们拔出钉子打开了棺材，看到德古拉正躺在里面。乔纳森立即拔出他的刀。但就在这时，太阳下山了。德古拉微笑着睁开眼睛，然后……

如果你对最终的结局感到好奇，可以点击 [查看这里](http://www.gutenberg.org/files/345/345-h/345-h.htm#CHAPTER_XIX) 并搜索“the look of hate in them turned to triumph”。

然而，我们可以确信的是女吸血鬼们已经被摧毁了，因此我们可以通过给她们一个 `last_appearance` 来做最后的改变。范海辛在 11 月 5 日摧毁了她们，因此我们将插入该日期。但不要忘记过滤掉露西 —— 她并不在居住在城堡里的三个女 `MinorVampire` 之中。

```edgeql
UPDATE MinorVampire FILTER .name != 'Lucy Westenra'
SET {
  last_appearance := <cal::local_date>'1887-11-05'
};
```

根据最后一场战斗中发生的事情，我们可能不得不对德古拉或一些英雄做同样的事情……

## 审查架构（Reviewing the schema）

[点击这里查看到目前为止我们搭建的架构和插入的所有数据](code.md)

现在你已经学习了 20 章，你应该对我们搭建的架构以及如何使用它已经有了很好的理解。让我们从上到下再看一遍，以确保我们完全理解了它，并考虑在实际游戏中哪些部分是好的，哪些还需要改进。

一个架构（a schema）的最初始终是启动迁移的命令：

- `START MIGRATION TO {};`：这就是架构迁移的开始方式。一切都在大括号 `{}` 内，并以分号 `;` 结尾。
- `module default {}`：在我们的架构里只使用了一个模块（命名空间），但如果你愿意，你可以制作更多。你可以使用 `DESCRIBE TYPE AS SDL`（或 `AS TEXT`）看到该模块。

这里有一个 `Person` 的例子，它像下面这样开始，并向我们显示了它所在的模块：

`abstract type default::Person`

对于真正的游戏，我们的架构可能会更大，包含各种模块。我们可能会看到各个类型在不同的模块里，比如 `abstract type characters::Person` 和 `abstract type places::Place`，甚至模块中的模块，比如 `type characters::PC::Fighter` 和 `type characters::NPC::Barkeeper`。

我们的第一个类型叫做 `HasNameAndCoffins`，它是抽象的，因为我们不想要任何这种类型的实际对象。相反，它被像 `Place` 这样的类型所扩展，因为在我们游戏中的每一个地方：

1. 都有一个名称；
2. 都有许多棺材（这很重要，因为没有棺材的地方对人类来说更安全，因为吸血鬼更难出没于此）。

```sdl
abstract type HasNameAndCoffins {
  required property coffins -> int16 {
    default := 0;
  }
  required property name -> str {
    constraint exclusive;
    constraint max_len_value(30);
  }
}
```

我们本可以对 `coffins` 属性使用 [`int32`、`int64` 或 `bigint`](https://www.edgedb.com/docs/datamodel/scalars/numeric#numerics)，但我们可能不会看到有那么多棺材，所以 `int16` 足够了。

接下来是 `abstract type Person`。这个类型是迄今为止最大的，并且为我们所有的角色完成了大部分的工作。幸运的是，所有吸血鬼曾经都是人，并且可以拥有诸如 `name` 和 `age` 之类的东西，因此它们也可以从 `Person` 扩展出来。

```sdl
abstract type Person {
  property first -> str;
  property last -> str;
  property title -> str;
  property degrees -> str;
  required property name -> str {
    constraint exclusive
  }
  property age -> int16;
  property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
  property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;
  property strength -> int16;
  multi link places_visited -> Place;
  multi link lover -> Person;
  property first_appearance -> cal::local_date;
  property last_appearance -> cal::local_date;
}
```

`exclusive` 可能是最常见的 [约束](https://www.edgedb.com/docs/datamodel/constraints#constraints) 了，我们用它来确保每个角色都有一个唯一的（无重复的）名称。这样就足够了，是因为我们已经知道所有 `NPC` 类型的名称没有重复的。但是，如果有可能出现多个“乔纳森·哈克（Jonathan Harker）”或其他角色的名称，我们则需要给 `Person` 一个 `id` 属性，并将其设为独占（`exclusive`）。

像 `conversational_name` 这样的属性是 [可计算的组件（computables）](https://www.edgedb.com/docs/datamodel/computables#computables)。在我们的例子中，我们稍后添加了诸如 `first` 和 `last` 之类的属性。删除 `name` 并只对每个角色使用 `first` 和 `last` 是不错，但书中有太多名字奇怪的角色，比如：`Woman 2`、`The innkeeper`等。在标准的用户数据库中，我们当然只会使用 `first` 和 `last` 以及带有 `constraint exclusive` 的 `email` 字段，以确保所有用户都是唯一的。

每个属性都有一个类型（如 `str`、`bigint` 等）。 可计算的组件（computables）也有，但我们不需要告诉 EdgeDB 要什么类型，因为它本身就构成了类型。例如，`pen_name` 用到了 `str` 类型的 `.name`，并添加更多其他的字符串，这当然会产生一个 `str`。其中用于将它们连接在一起的 `++` 称为 [拼接/串联（concatenation）](https://www.edgedb.com/docs/edgeql/funcops/string#operator::STRPLUS)。

其中有两个链接是 `multi link`，如果没有 `multi`，一个 `link` 只能指向一个对象。如果你仅写 `link`，它将是一个 `single link`，你必须在创建链接时添加`LIMIT 1`，否则会出现这个错误：

```
error: possibly more than one element returned by an expression for a computable link 'former_self' declared as 'single'
```

对于 `first_appearance` 和 `last_appearance`，我们使用 `cal::local_date`，因为我们的游戏设定在特定时期内且仅基于欧洲的一部分。对于现代的用户数据库，我们更喜欢 [`std::datetime`](https://www.edgedb.com/docs/datamodel/scalars/datetime#type::std::datetime) 因为它是感知时区的并且总是符合 ISO8601。

所以对于用户遍布全球的数据库来说，`datetime` 通常是最好的选择。然后，你可以使用 [`std::to_datetime`](https://www.edgedb.com/docs/edgeql/funcops/datetime#function::std::to_datetime) 之类的函数将五个 `int64`、一个 `float64`（用于秒）和一个 `str`（用于 [the timezone](https://en.wikipedia.org/wiki/List_of_time_zone_abbreviations)）转换为一个总是作为 UTC 返回的 `datetime`：

```edgeql-repl
edgedb> SELECT std::to_datetime(2020, 10, 12, 15, 35, 5.5, 'KST');
....... # October 12 2020, 3:35 pm and 5.5 seconds in Korea (KST = Korean Standard Time)
{<datetime>'2020-10-12T06:35:05.500000000Z'} # The return value is UTC, 6:35 (plus 5.5 seconds) in the morning
```

还有一个与 `HasNameAndCoffins` 类似的抽象类型是：

```sdl
abstract type HasNumber {
  required property number -> int16;
}
```

我们只将它用于 `Crewman` 类型，`Crewman` 扩展自两个抽象类型：

```sdl
type Crewman extending HasNumber, Person {
}
```

这个 `HasNumber` 类型用于五个 `Crewman` 对象，它们一开始没有名字。但后来，我们使用这些数字为他们创建了基于数字名称：

```edgeql
UPDATE Crewman
SET {
  name := 'Crewman ' ++ <str>.number
};
```

所以即使它很少被用到，它也可能在之后会变得有用。对于游戏后期的类型，你可以想象这将被用于居民或随机 NPC：“店主 2”、“马车司机 12”等等。

我们的吸血鬼类型扩展了 `Person`，而 `MinorVampire` 也有一个到 `Person` 的可选（单一）链接。这是因为一些角色最初是人类，然后“重生”成为了吸血鬼。有了这个格式，我们可以使用 `Person` 中的 `first_appearance` 和 `last_appearance` 属性让各个角色出现在游戏中。如果有人变成了 `MinorVampire`，我们可以将两者链接起来。

```sdl
type Vampire extending Person {
  multi link slaves -> MinorVampire;
}

type MinorVampire extending Person {
  link former_self -> Person;
}
```

有了这个格式，我们可以像下面这样查询所有变成 `MinorVampire` 的人。

```edgeql
SELECT Person {
  name,
  vampire_name := .<former_self[IS MinorVampire].name
} FILTER EXISTS .vampire_name;
```

在我们的例子中，只有 Lucy: `{default::NPC {name: 'Lucy Westenra', vampire_name: {'Lucy Westenra'}}}` 但如果我们愿意，我们可以将游戏扩展到更早的历史时期，并将那三个吸血鬼女性与 `NPC` 类型联系起来。那将成为他们的 `former_self`。

我们的两个枚举用于 `PC` 和 `Sailor` 类型：

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
type Sailor extending Person {
  property rank -> Rank;
}

scalar type Transport extending enum<Feet, HorseDrawnCarriage, Train>;
type PC extending Person {
  required property transport -> Transport;
}
```

枚举 `Transport` 从未真正被使用过，需要更多的运输类型。我们没有详细研究这些，但在书中有很多不同类型的传输方式。例如，在最后一章中，在瓦尔纳（Varna）等候的亚瑟（Arthur）的团队使用了一艘名为“蒸汽发射（steam launch）”的船，该船比“德米特（The Demeter）”号还小。这个枚举可能会以下面的方式用于游戏逻辑本身：

- 选择 `Feet` 会给角色一定的速度，而且不需要任何费用，
- `HorseDrawnCarriage` 提高速度但会减少金钱，
- `Train` 提速最多，同样会减少钱，且只能沿着铁路线行驶等。

`Visit` 是我们两种“最黑客”（但最有趣）的类型之一。我们从我们之前创建但从未使用过的 `Time` 类型中窃取了大部分。在其中，我们有一个 `time` 属性，它只是一个字符串，以下面的方式使用：

- 通过将其转换为 <cal::local_time> 以创建 `local_time` 属性，
- 通过切片它的前两个字符来获得 `hour` 属性，它只是一个字符串。这是唯一可能的，因为我们知道即使像 `1` 这样的单个数字也需要用两位数书写：“01”，
- 由另一个名为 `awake` 的可计算性决定是 'sleep' 或 'awake'，具体取决于我们刚刚创建的 `hour` 属性，并转换为 `int16`。

```sdl
type Visit {
  required link ship -> Ship;
  required link place -> Place;
  required property date -> cal::local_date;
  property time -> str;
  property local_time := <cal::local_time>.time;
  property hour := .time[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
```

NPC 类型是我们第一次看到 [`overloaded`](https://www.edgedb.com/docs/edgeql/sdl/links#overloading) 关键字的地方，它让我们可以以不同于默认的方式使用属性、链接、函数等。在这里，我们希望将 `age` 限制为 120 岁，并以不同于 `Person` 的方式使用 `places_visited` 链接，将其默认值设置为 `London`。

```sdl
type NPC extending Person {
  overloaded property age {
    constraint max_value(120)
  }
  overloaded multi link places_visited -> Place {
    default := (SELECT City FILTER .name = 'London');
  }
}
```

我们的 `Place` 类型显示出你可以根据需要扩展任意多次。它是一个 `abstract type`，它扩展自另一个 `abstract type`，然后再扩展为其他类型，例如 `City`。

```sdl
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

`important_places` 属性仅在此插入中使用过一次：

```edgeql
INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

`important_places` 现在只是一个数组。我们现在可以保持不变，因为我们还没有为酒店和公园等非常小的地方创建类型。但是如果我们确实为这些地方创建了一个新类型，那么我们应该把它变成一个“多链接”。甚至我们的 `OtherPlace` 类型也不是完全正确的类型，如 [annotation](https://www.edgedb.com/docs/edgeql/sdl/annotations#annotations) 所示：

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns count! Use the Castle type for that';
}
```

因此，在真实的游戏中，我们会创建一些其他较小的位置类型，并将它们设为来自 `City` 内的 `property important_places` 属性的链接。我们也可以将 `important_places` 移动到 `Place`，这样像 `Region` 这样的类型也可以从中链接。

注释：我们使用 `abstract annotation` 来添加新注释：

```sdl
abstract annotation warning;
```

因为默认情况下，类型只能有称为 `title`、`description` 或 `deprecated` 的 [注释](https://www.edgedb.com/docs/datamodel/annotations#ref-datamodel-annotations)。我们对这种类型使用注释只是为了好玩，因为还没有其他人同时在处理我们的数据库。但是，如果我们为一个有很多人参与制作的游戏创建了一个真实的数据库，我们就会在各处放置注释，以确保所有人都知道该如何使用每种类型。

我们创建 `Lord` 类型只是为了展示如何使用 `constraint expression on`，这让我们可以创建我们自己的约束：

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord') = true) {
    errmessage := "All lords need \'Lord\' in their name";
  };
};
```

（我们可能会在真正的游戏中删除它，或者它可能会成为扩展自 PC 的类型 Lord，以便玩家角色可以选择成为领主、小偷、或侦探等等。）

`Lord` 类型使用了函数 [`contains`](https://www.edgedb.com/docs/edgeql/funcops/generic#function::std::contains)，如果字符串、数组等包含我们正在搜索的项目，则该函数返回 `true`。这里还使用了 `__subject__`，它指的是类型本身：在这种情况下，`__subject__.name` 的意思是 `Person.name`。[这里有更多示例](https://www.edgedb.com/docs/datamodel/constraints#constraint::std::expression) 来自 `constraint expression on` 的使用文档。

另一种创建 `Lord` 的可能方法如下所示，因为 `Person` 具有名为 `title` 的属性：

```sdl
type Lord extending Person {
  constraint expression on (__subject__.title = 'Lord') {
    errmessage := "All lords need \'Lord\' in their name";
  };
}
```

这将取决于我们创建 `Lord` 类型时，名称是使用 `.name` 中的单个字符串，还是使用带有可计算形式的 `.first`、`.last`、`.title` 等全名。

接下来扩展自 `Place` 的类型包括 `Country` 和 `Region`，我们刚在上一章看到过，所以不在这里回顾它们了。但是 `Castle` 有些特别：

```sdl
type Castle extending Place {
  property doors -> array<int16>;
}
```

回到第 7 章，我们在查询中使用了它来查看乔纳森（Jonathan）是否可以打破任何门并逃离城堡。这个想法很简单：乔纳森会尝试打开每一扇门，只要他比其中任意一个门更有力量，那么他就可以逃离城堡。

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT jonathan_strength > min(array_unpack(doors));
```

但是，稍后我们学习了 `any()` 函数，所以让我们看看如何在这里使用它。使用 `any()`，我们可以将查询更改为：

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT any(array_unpack(doors) < jonathan_strength); # Only this part is different
```

当然，我们也可以创建一个函数来做同样的事情，因为我们知道如何编写函数以及如何使用 `any()`。由于我们按名称进行过滤（Jonathan Harker 和 Castle Dracula），因此该函数也将只采用两个字符串并执行相同的查询。

不要忘记，我们还需要 `array_unpack()`，因为函数 [`any()`](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::any) 是作用于集合的：
Don't forget, we needed `array_unpack()` because the function [`any()`](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::any) works on sets:

```sdl
std::any(values: SET OF bool) -> bool
```

所以这个（一个集合）将起作用：`SELECT any({5, 6, 7} = 7);`

但是这个（一个数组）将不工作：`SELECT any([5, 6, 7] = 7);`

接下来的类型是 `BookExcerpt`，我们认为它对创建数据库的人是有用的。它需要从书中的每个部分插入大量内容，并且文本与所写的完全相同。因此，我们选择使用 [`index on`](https://www.edgedb.com/docs/edgeql/sdl/indexes#indexes) 作为 `excerpt` 属性，这样查找起来会更快。记住仅在需要的地方使用它：它会提高查找速度，但也会使数据库整体变大。

```sdl
type BookExcerpt {
  required property date -> cal::local_datetime;
  required link author -> Person;
  required property excerpt -> str;
  index on (.excerpt);
}
```

接下来是另一个有趣且“黑客”的类型 `Event`。

```sdl
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  multi link excerpt -> BookExcerpt;
  property exact_location -> tuple<float64, float64>;
  property east -> bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ 'E' if .east = true else 'W';
}
```

这个类型可能最接近真实游戏的实际可用的类型。使用 `start_time` 和 `end_time`、`place` 和 `people`（加上 `url`），我们可以正确地安排什么角色在什么位置以及何时。`description` 属性是为数据库用户提供的，如 `'The Demeter arrives at Whitby, crashing on the beach'` 这样的描述，用于在我们需要时查找事件。

架构中的最后的两种类型是 `Currency` 和 `Pound`，它们是在两章前刚创建的，所以我们就不在这里回顾它们了。

## 导览 EdgeDB 文档

现在你已经读到了本书的结尾，你肯定会开始查看 EdgeDB 文档。我们将通过一些提示来结束本书，以便你查看文档时感觉熟悉且易于阅读。

### 句法

本书包含了很多 EdgeDB 文档的链接，例如类型、函数等。如果你尝试创建类型、属性等并且遇到问题，最好从语法部分开始。这部分展示了需要遵循的顺序以及你拥有的所有选项。

举个简单的例子，[这里是创建模块的语法](https://www.edgedb.com/docs/edgeql/sdl/modules)：

```sdl-synopsis
module ModuleName "{"
  [ schema-declarations ]
  ...
"}"
```

你可以看到一个模块只是一个模块名称加上 `{}` 和里面的所有东西（架构声明）。这很容易。

那么，对象类型呢？ [它们看起来像这样](https://www.edgedb.com/docs/edgeql/sdl/objects)：

```sdl-synopsis
[abstract] type TypeName [extending supertype [, ...] ]
[ "{"
    [ annotation-declarations ]
    [ property-declarations ]
    [ link-declarations ]
    [ constraint-declarations ]
    [ index-declarations ]
    ...
  "}" ]
```

这对你来说应该很熟悉了：你需要使用 `type TypeName` 来启动。你可以在左侧添加 `abstract`，也可以在右侧添加 `extending` 来拓展某个类型，然后其他所有内容都放在随后的 `{}` 当中。

同时，[属性更复杂些](https://www.edgedb.com/docs/edgeql/sdl/props) 它包括三种类型：具体的（concrete）、可计算（computable）的和抽象的（abstract）。让我们来看一下我们最熟悉的“具体的（concrete）”属性声明：

```sdl-synopsis
[ overloaded ] [{required | optional}] [{single | multi}]
  property name
  [ extending base [, ...] ] -> type
  [ "{"
      [ default := expression ; ]
      [ readonly := {true | false} ; ]
      [ annotation-declarations ]
      [ constraint-declarations ]
      ...
    "}" ]
```

你可以将语法视为有助于使声明保持正确顺序的指南。

### 深入探寻 DDL

DDL 是你经常会在文档中看到的内容。到目前为止，我们只提到了函数的 DDL，因为在需要时添加 `CREATE` 来创建函数非常容易。

SDL: `function says_hi() -> str using('hi');`

DDL: `CREATE FUNCTION says_hi() -> str USING('hi')`

大小写也无关紧要。

但是对于类型，DDL 需要更多的输入，使用诸如 `CREATE`、`SET`、`ALTER` 等关键字。如果你想进行小的更改或快速键入，这仍然是值得的。例如，这是一个只有名字的 Cat 类型：

```sdl
type Cat {
  property name -> str;
  property sound := 'meow';
};
```

当你输入 `DESCRIBE TYPE Cat as SDL` 时，它会显示类似的内容：

```sdl
type default::Cat {
  optional single property cat_name -> std::str;
};
```

这是相同的声明，但包括一些我们不需要指定的信息：

- 它在模块 default 中，
- 它是可选的，而不是必需的，
- 它是一个单一的属性，而不是多个。

那么它作为 DDL 是什么样子的呢？ `DESCRIBE TYPE Cat` 将向我们展示：

```edgeql
CREATE TYPE default::Cat {
  CREATE OPTIONAL SINGLE PROPERTY name -> std::str;
};
```

没有你想象的那么糟糕！使用 DDL 的两个一般规则是：

- 只需在每一行添加 `CREATE` 你就可以完成很多工作，用 `ALTER` 更改内容，用 `DROP` 删除它们，
- 使用 `DESCRIBE TYPE` 和 `DESCRIBE TYPE AS SDL` 来比较两者非常有用。如果使用你创建并熟悉的类型执行此操作，你将可以使用它作为垫脚石如果你对 DDL 好奇的话。

## EdgeDB 词法结构

你可能想要查看或收藏 [此页面](https://www.edgedb.com/docs/edgeql/lexical/) 以供你在项目期间可以参考。它包含 EdgeDB 的整个词汇结构，包括一些可能对这样的教科书来说有些枯燥的条目。诸如运算符的优先顺序、所有保留关键字、可在标识符中使用哪些字符等等。

## 获得帮助

帮助总是一条信息就能搞定。获得帮助的最佳方式是在 GitHub 上的 [我们的讨论板](https://github.com/edgedb/edgedb/discussions) 上发起讨论。你还可以对 EdgeDB [在此处提出问题](https://github.com/edgedb/edgedb/issues/new/choose)，也同样可以在该页面对本书提出问题。

## 再见

我们希望你能喜欢通过这个故事来学习 EdgeDB，并且现在已经足够熟悉 EdgeDB 以在你自己的项目中使用它。有趣的是，如果我们写的书足够详细以回答你的所有问题，那么我们可能永远不会在论坛上看到你！如果是那样，我们只能祝你的项目一切顺利。现在，让我们用另一本书《指环王》中的一首诗来结束这本书，关于生命的无限可能性。

> The Road goes ever on and on
> 漫漫长路
> 
> Down from the door where it began.
> 起于家门
> 
> Now far ahead the Road has gone,
> 路其修远
> 
> And I must follow, if I can,
> 紧随不息
> 
> Pursuing it with eager feet,
> 步履匆匆
> 
> Until it joins some larger way
> 直奔通途
> 
> Where many paths and errands meet.
> 歧路迭起
> 
> And whither then?
> 去向何方？
> 
> I cannot say.
> 我心彷徨
> 
> PS: 中文翻译部分采用石中歌老师，邓嘉宛老师和杜蕴慈老师译本。

再见，或者不再见，不管结果如何！再次感谢你的阅读。
