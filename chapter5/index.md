---
tags: Datetime, Describing Types
leadImage: illustration_05.jpg
---

# 第五章 - 乔纳森试图离开城堡

可怜的乔纳森（Jonathan）运气不佳，在这一章里他发生了什么？

> 白天，乔纳森决定尝试探索这座城堡，但太多的门窗都被锁上了。他不知道怎么出去，希望至少能给米娜寄一封信。他假装没有问题，在夜里继续不停地和德古拉聊天。一天晚上，他看到德古拉爬出窗口，像一条蛇一样爬下城墙。他很害怕，知道了德古拉应该不是人类。几天后，他打破了一扇门，发现了城堡的另一部分。这个房间很奇怪，让他开始犯困。当他睁开眼睛时，他看到了三个吸血鬼女人在他的身边。他既被她们吸引又害怕她们。他想亲吻她们，但他知道如果他这样做了就会死。她们靠得更近了，但他无法动弹……

## 日期时间（std::datetime）

由于乔纳森（Jonathan）想到了回到伦敦的米娜（Mina），让我们了解一下 `std::datetime`，因为它使用时区。要创建日期时间，您只需使用 `<datetime>` 转换为 ISO 8601 格式的字符串。该格式如下所示：

`YYYY-MM-DDTHH:MM:SSZ`

实际日期看起来像这样：

`'2020-12-06T22:12:10Z'`

其中 `T` 只是一个分隔符，最后的 `Z` 代表“零时间线（zero timeline）”。这意味着它与 UTC 有 0° 的偏移：换句话说，它_就是_UTC。

获取 `datetime` 的另一种方法是使用 `to_datetime()` 函数。[这里是它的函数签名](https://edgedb.com/docs/edgeql/funcops/datetime/#function::std::to_datetime)，里面有六种方法可以通过使用此函数生成 `datetime`，具体取决于你想要如何生成。EdgeDB 将根据你提供的输入得知你选择了六种中的哪一种。

顺便说一下，你可能会留意到一个不太熟悉的、名为 [`decimal`](https://www.edgedb.com/docs/datamodel/scalars/numeric#type::std::decimal) 的类型。这是一个具有“任意精度”的浮点数，这意味着你可以根据需要在小数点后给出任意数量的数字。计算机上的浮点类型会由于舍入错误 [一段时间后会变得不精确](https://www.youtube.com/watch?v=-3c8G0JMM5Q)。比如下面的例子：

```edgeql-repl
edgedb> SELECT 6.777777777777777; # Good so far
{6.777777777777777}
edgedb> SELECT 6.7777777777777777; # Add one more digit...
{6.777777777777778} # Where did the 8 come from?!
```

如果你想避免这个问题，在末尾添加一个 `n` 以获得一个 `decimal` 类型，它将尽可能精确。

```edgeql-repl
edgedb> SELECT 6.7777777777777777n;
{6.7777777777777777n}
edgedb> SELECT 6.7777777777777777777777777777777777777777777777777n;
{6.7777777777777777777777777777777777777777777777777n}
```

同时，还有一个 `bigint` 类型，也可以使用 `n` 来表示任意大小。那是因为即使 int64 也有上限：它是 9223372036854775807。

```edgeql-repl
edgedb> SELECT 9223372036854775807; # Good so far...
{9223372036854775807}
edgedb> SELECT 9223372036854775808; # But add 1 and it will fail
ERROR: NumericOutOfRangeError: std::int64 out of range
```

所以这里你可以加上一个 `n`，它会创建一个可以容纳任何大小的 `bigint`。

```edgeql-repl
edgedb> SELECT 9223372036854775808n;
{9223372036854775808n}
```

现在我们知道了所有的数字类型，让我们回到 `std::to_datetime` 函数的六个签名：

```
std::to_datetime(s: str, fmt: OPTIONAL str = {}) -> datetime
std::to_datetime(local: cal::local_datetime, zone: str) -> datetime
std::to_datetime(year: int64, month: int64, day: int64, hour: int64, min: int64, sec: float64, timezone: str) -> datetime
std::to_datetime(epochseconds: decimal) -> datetime
std::to_datetime(epochseconds: float64) -> datetime
std::to_datetime(epochseconds: int64) -> datetime
```

如果你对 ISO 8601 不熟悉，或者你有一堆单独的数字要组成一个日期，那么最简单的方法可能是第三个。有了这个，我们可以对“时间”里的整数通过使用 `to_datetime()` 来获得其对应的正确的时间戳。

假设现在是 5 月 12 日（May 12）10:35，在德古拉城堡中的一个晴朗的早上。太阳升起，德古拉在某处睡着，乔纳森正试图利用白天的时间逃出去给米娜寄一封信。在罗马尼亚，时区是“EEST”（东欧夏令时）。我们将使用 `to_datetime()` 来生成它。因为整个故事都发生在同一年，所以我们可以随便指定一个年份 —— 为了方便起见，我们将使用 2020 年。输入：

`SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST');`

并得到以下输出：

`{<datetime>'2020-05-12T07:35:00Z'}`

`07:35:00` 部分说明时间已自动转换为 UTC，即米娜居住的伦敦。

我们还可以使用它来查看事件之间的持续时间。EdgeDB 有一个 `duration` 类型，你可以通过用一个日期时间减去另一个日期时间来获得。现在让我们练习一下，计算一个中欧日期和一个韩国日期之间相差的确切秒数：

```edgeql
SELECT to_datetime(2020, 5, 12, 6, 10, 0, 'CET') - to_datetime(2000, 5, 12, 6, 10, 0, 'KST');
```

中欧时间 2020 年 5 月 12 日上午 6:10 减去韩国标准时间 2000 年 5 月 12 日 6:10。结果是：`{631180800s}`。

现在让我们再次尝试做与德古拉城堡中的乔纳森类似的事情 —— 试图逃脱。现在是 5 月 12 日上午 10:35。同一天，伦敦的早上 6 点 10 分，米娜正在喝着早茶。这两个事件之间相差多少秒？它们处于不同的时区，但我们不需要自己计算；我们可以只指定时区，EdgeDB 将完成剩下的工作：

```edgeql
SELECT to_datetime(2020, 5, 12, 10, 35, 0, 'EEST') - to_datetime(2020, 5, 12, 6, 10, 0, 'UTC');
```

答案是 5100 秒：`{5100s}`。

为了使查询语句更加易懂，我们还可以使用关键字 `WITH` 来创建变量。然后我们可以在下面的 `SELECT` 中使用这个变量。我们将创建两个变量，分别叫做 `jonathan_wants_to_escape` 和 `mina_has_tea`，然后从其中一个中减去另一个以获得 `duration`。有了变量名，我们现在要做的事情就显得更加清楚了：

```edgeql
WITH
  jonathan_wants_to_escape := to_datetime(2020, 5, 12, 10, 35, 0, 'EEST'),
  mina_has_tea := to_datetime(2020, 5, 12, 6, 10, 0, 'UTC'),
SELECT jonathan_wants_to_escape - mina_has_tea;
```

输出结果是一样的：`{5100s}`。只要我们知道时区，当我们需要一个 `duration` 时，`datetime` 类型就可以胜任。

## 强制转换为 duration（Casting to a duration）

除了对两个 `datetime` 进行相减，你也可以直接转换出一个 `duration`。为此，只需写下数字及对应的单位：`microseconds`, `milliseconds`, `seconds`, `minutes`, 或 `hours`（“微秒”、“毫秒”、“秒”、“分钟”或“小时”）。它将返回一个秒数或更精确的单位。比如 `SELECT <duration>'2 hours';` 将返回 `{7200s}`；`SELECT <duration>'2 microseconds';` 将返回 `{2µs}`。

你也可以包含多个单位。例如：

```edgeql
SELECT <duration>'6 hours 6 minutes 10 milliseconds 678999 microseconds';
```

这将返回：`{21960.688999s}`。

在做 `duration` 的转换时，EdgeDB 在输入方面是非常宽容，并且会忽略复数和其他符号。因此，即使是这种可怕的输入也可以工作（但我们并不推荐这样做）：

```edgeql
SELECT <duration>'1 hours, 8 minute ** 5 second ()()()( //// 6 milliseconds' -
  <duration>'10 microsecond 7 minutes %%%%%%% 10 seconds 5 hour';
```

结果是：`{-14344.99401s}`。

## 必需链接（Required links）

现在我们需要为三个女性吸血鬼创建一个类型。我们称它为 `MinorVampire`。它有一个 `required` 的、指向 `Vampire` 类型的链接。这是因为她们被德古拉控制着，她们只因为德古拉的存在而作为 `MinorVampire` （小鬼）存在。

```sdl
type MinorVampire extending Person {
  required link master -> Vampire;
}
```

现在 `master` 是必需的，我们不能插入一个只有名字的 `MinorVampire`。如果那样做，我们会得到错误：`ERROR: MissingRequiredError: missing value for required link default::MinorVampire.master`。因此让我们插入数据并将其连接到德古拉（Dracula）。

```edgeql
INSERT MinorVampire {
  name := 'Woman 1',
  master := (SELECT Vampire Filter .name = 'Count Dracula'),
};
```

上面之所以可以工作，是因为我们的数据库里只有一个名为德古拉伯爵（Count Dracula）的 `Vampire`（请记住，`required link` 是 `required single link` 的缩写）。如果我们的数据里有不止一个 `Vampire`，我们则必须加上 `LIMIT 1`，否则我们会得到如下的报错：

```
error: possibly more than one element returned by an expression for a computable link 'master' declared as 'single'
```

## 查看内部类型（DESCRIBE to look inside types）

`MinorVampire` 类型扩展自 `Person`，`Vampire` 也是如此。类型可以继续扩展其他类型，并且可以同时扩展多个类型。你这样做的次数越多，试图在脑海中将它们组合在一起就越困难。这正是 `DESCRIBE` 可以提供帮助的地方，它可以准确地显示出任何类型的组成内容。具体有以下三种方法：

- `DESCRIBE TYPE MinorVampire` —— 这将给出类型的 [DDL（数据定义语言）](https://www.edgedb.com/docs/edgeql/ddl/index/)描述。
DDL 是比 SDL（我们一直在使用的语言）更低级别的语言。它对 schema 不太方便，但更明确，可用于快速更改。我们不会在本课程中系统学习 DDL，但稍后你可能会发现它有时很有用。例如，使用它你可以快速创建函数而无需进行 _显式的（explicit）_ 迁移（migration，这里的 migration 指的是一次完整的、常规的 schema 变更）。

(请注意上面提到的 _显式的（explicit）_ 一词：使用 DDL 仍会导致迁移，只是 _隐式的（implicit）_ 迁移。换句话说，迁移发生时并未将其称为迁移。这是一种快又脏的更改方式，但在大多数情况下，使用 SDL schema 的适当迁移工具是首选方式。）

现在让我们回到 `DESCRIBE TYPE` 用 DDL 给出的结果。下面是 `Person` 类型的样子：

```
{
  'CREATE TYPE default::MinorVampire EXTENDING default::Person {
    CREATE REQUIRED SINGLE LINK master -> default::Vampire;
  };',
}
```

`CREATE` 关键字表明它是一系列的快速命令，这也是顺序很重要的原因。换句话说，SDL 是 _陈述的（declarative）_（它 _陈述_ 将是什么，而不用担心顺序），而 DDL 是 _命令的（imperative）_（它是一系列更改状态的命令）。此外，它只显示了创建它的 DDL 命令，没有向我们显示它扩展的所有 `Person` 的链接和属性，这不是我们想要的。下一个方法是：

- `DESCRIBE TYPE MinorVampire AS SDL` - 同样的事情，但用 SDL 表达。

输出也几乎相同，只是上一个方法输出结果的 SDL 版本。对于我们现在想要的信息，这也不够：

```
{
  'type default::MinorVampire extending default::Person {
    required single link master -> default::Vampire;
  };',
}
```

你会注意到它与我们的 SDL 架构（schema）基本相同，只是更加冗长和详细：`type default::MinorVampire` 替代了 `type MinorVampire` 等等。

第三个方法是 `DESCRIBE TYPE MinorVampire AS TEXT`。这就是我们想要的，因为它显示了类型内部的所有内容，包括它所扩展的类型的。这是输出：

```
{
  'type default::MinorVampire extending default::Vampire {
    required single link __type__ -> schema::Type {
        readonly := true;
    };
    optional single link lover -> default::Person;
    required single link master -> default::Vampire;
    optional multi link places_visited -> default::Place;
    optional single property age -> std::int16;
    required single property id -> std::uuid {
        readonly := true;
    };
    required single property name -> std::str;
  };',
}
```

（请注意：`AS TEXT` 不包括约束和注释。要查看这些，请在末尾添加 `VERBOSE`：`DESCRIBE TYPE MinorVampire AS TEXT VERBOSE;`。你将在第 14 章中学习注释相关。）

`readonly := true` 的部分我们不必在意，它们是自动生成的（我们无法对它们做什么）。对于其他部分，我们可以看到我们必须有一个 `name` 和一个 `master`，且可以选择为这些 `MinorVampire` 添加 `lover`、`age` 和 `places_visited`。

对于 _真的_ 长输出，请尝试键入 `DESCRIBE SCHEMA` 或 `DESCRIBE MODULE default`（如果需要，可以使用 `AS SDL` 或 `AS TEXT` ）。你将获得迄今为止我们构建的整个架构的输出。

因此，对于类型，我们把 `TYPE` 放到 `DESCRIBE` 后面；对于模块，我们把 `MODULE` 放到 `DESCRIBE` 后面，那么对于链接或其他的呢？以下是可以放到 `DESCRIBE` 后面的所有关键字的列表：`OBJECT`、`ANNOTATION`、`CONSTRAINT`、`FUNCTION`、`LINK`、`MODULE`、`PROPERTY`、`SCALAR TYPE`、`TYPE`。如果你不想全部记住它们，只需使用 `OBJECT`：它会匹配你的架构（schema）中的任何内容（模块除外）。

[→ 点击这里查看第 5 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 你认为 `SELECT to_datetime(3600);` 将返回什么？为什么？

   提示：检查上面的函数签名，看看当你输入 3600 时，EdgeDB 会选择哪一个。

2. `SELECT <int16>9 + 1.06n IS decimal;` 能工作吗？如果可以，它将返回 `{true}` 吗？

3. 从 2003 年土库曼斯坦 (TMT) 圣诞节的早上 5:00 到乌兹别克斯坦 (UZT) 同年新年除夕夜的晚上 7:00 之间过去了多少秒？

4. 如何用对上题中的两个时间 `WITH` 写出同样查询效果的查询语句？

5. 如果您只想查看如何编写的某个类型，那么描述该类型的最佳方式是什么？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _一位女吸血鬼对她的姐妹们说：“他年轻且强壮；我们所有人都有亲吻……”_
