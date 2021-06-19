---
tags: Tuples, Computables, Math
---

# 第十章 - 惠特比的可怕事件

> 米娜（Mina）和露西（Lucy）正在惠特比（Whitby）享受他们愉快的时光。一天晚一场大风暴来袭，一艘船在雾中靠岸 —— 正是德古拉所在的德米特号（the Demeter）。露西后来开始在晚上梦游，脸色苍白，一直说些奇怪的话。米娜试图阻止她，但有时露西会跑到外面去。一天晚上，露西看着太阳下山说：“他的眼睛又红了！他们一样。”米娜很担心，向苏厄德医生（Dr. Seward）寻求帮助。苏厄德医生对露西进行了检查，露西脸色苍白，身体虚弱，但他不知道为什么。苏厄德医生决定打电话给他来自荷兰的老师亚伯拉罕·范海辛 (Abraham Van Helsing) 寻求帮助。范海辛对露西进行了检查，他很震惊。然后他转向其他人说：“听着。我们可以帮助这个女孩，但你们会觉得方法很奇怪。你们必须相信我……”

惠特比市（Whitby）位于英格兰（England）东北部。现在我们的 `City` 类型只是扩展了 `Place`，它只给了我们属性 `name`、`modern_name` 和 `important_places`。这可能是增加 `property population`（人口属性）的好时机，它可以帮助我们在游戏具象一个城市。我们将使用 `int64` 来表达它的大小：

```sdl
type City extending Place {
  property population -> int64;
}
```

顺便说一下，这是本书出版时相关城市的大致人口。这些城市的人口在 1887 年时还很少：

- Buda-Pesth (Budapest): 402706
- London: 3500000
- Munich: 230023
- Whitby: 14400
- Bistritz (Bistrița): 9100

插入惠特比市（Whitby）很容易：
Inserting Whitby is easy enough:

```edgeql
INSERT City {
  name := 'Whitby',
  population := 14400
};
```

但是对于其他几个城市来说，同时更新所有内容会很好。

## 元组和数组的使用（Working with tuples and arrays）

如果我们将所有城市数据放在一起，我们可以再次使用 `FOR` 和 `UNION` 循环进行一次单次插入（或更新）。假设我们在“元组（tuples）”中有一些数据，它看起来与数组（arrays）相似，但又完全不同。一个很大的区别是元组可以包含不同的类型，所以下面这样是可以的：

`('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)`

在这种情况下，该类型称为 `tuple<str, int64>`。

在我们开始使用这些元组之前，让我们确保我们了解了元组（tuples）与数组（arrays）之间的区别。首先，让我们更详细地了解切片数组（slicing arrays）和字符串（strings）。

你一定记得我们使用方括号来访问数组或字符串的一部分。所以 `SELECT ['Mina Murray', 'Lucy Westenra'][1];` 将给出输出 `{'Lucy Westenra'}`（即索引 1 对应的数值）。

你也应该还记得，我们可以用冒号分隔起始索引和结束索引，如下例所示：

```edgeql
SELECT NPC.name[0:10];
```

这将打印每个 NPC 名字的前十个字母：

```
{
  'Jonathan H',
  'The innkee',
  'Mina Murra',
  'John Sewar',
  'Quincey Mo',
  'Lucy Weste',
  'Arthur Hol',
}
```

如果你想从最后的索引开始，同样可以用负数来完成。例如：

```edgeql
SELECT NPC.name[2:-2];
```

这将从索引 2 打印到距离末尾 2 个索引（即切断每一侧的前两个字母）的所有字母。这是输出：

```
{
  'nathan Hark',
  'e innkeep',
  'na Murr',
  'hn Sewa',
  'incey Morr',
  'cy Westen',
  'thur Holmwo',
}
```

元组完全不同：它们的行为更像是具有数字而不是名称的属性的对象类型。这就是元组可以将不同类型保存在一起的原因：`string` 和 `array`，`int64` 和 `float32`，等等。

所以下面这个完全没问题：

```edgeql
SELECT {('Bistritz', 9100, cal::to_local_date(1887, 5, 6)), ('Munich', 230023, cal::to_local_date(1887, 5, 8))};
```

这是输出：

```
{
  ('Bistritz', 9100, <cal::local_date>'1887-05-06'),
  ('Munich', 230023, <cal::local_date>'1887-05-08'),
}
```

但是现在类型已设置（上面输出的是类型 `tuple<str, int64, cal::local_date>`），你不能将它与其他元组类型混淆。像下面这样是不被允许的：

```edgeql
SELECT {(1, 2, 3), (4, 5, '6')};
```

EdgeDB 会报错，因为它不会尝试处理不同类型的元组。错误提示是：
EdgeDB will give an error because it won't try to work with tuples that are of different types. It complains:

```
ERROR: QueryError: operator 'UNION' cannot be applied to operands of type 'tuple<std::int64, std::int64, std::int64>' and 'tuple<std::int64, std::int64, std::str>'
  Hint: Consider using an explicit type cast or a conversion function.
```

在上面的例子中，我们可以轻松地将第二个元祖中的最后一个字符串转换为整数，EdgeDB 则会再次工作：`SELECT {(1, 2, 3), (4, 5, <int64>'6')};`。

要访问元组的字段，仍然从是数字 0 开始，但将数字写在 `.` 之后，而不是在 `[]` 内。现在我们已经了解了这一切，我们可以同时更新我们所有的城市了。它看起来像这样：

```edgeql
FOR data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
UNION (
  UPDATE City FILTER .name = data.0
  SET {
    population := data.1
  }
);
```

因此，它将每个元组发送到 `FOR` 循环中，按名字的字符串（即 `data.0`）进行过滤，然后对人口（即 `data.1`）进行更新。

在这个部分的最后，我们再说一下关于类型转换。我们知道对于任何标量类型我们都可以做类型转换，这也同样适用于标量类型的元组。它使用相同的格式 `<>`，只是需要把它放在 `<tuple>` 里面，像这样：

```
WITH london := ('London', 3500000),
SELECT <tuple<json, int32>>london;
```

这是输出：

```
{("\"London\"", 3500000)}
```

这里是另一个例子，如果我们需要对伦敦人口进行一些浮点数的数学计算：

```
WITH london := <tuple<json, float64>>('London', 3500000),
  SELECT (london.0, london.1 / 9);
```

输出是：`{("\"London\"", 388888.8888888889)}`.

## 排序和数学（Ordering results and using math）

现在我们有了一些数字，我们可以开始玩排序和数学了。排序是非常简单的：输入 `ORDER BY`，然后指明你要排序的属性/链接。这里我们按人口排序：

```edgeql
SELECT City {
  name,
  population
} ORDER BY .population DESC;
```

这是结果：

```
{
  Object {name: 'London', population: 3500000},
  Object {name: 'Buda-Pesth', population: 402706},
  Object {name: 'Munich', population: 230023},
  Object {name: 'Whitby', population: 14400},
  Object {name: 'Bistritz', population: 9100},
}
```

什么是 `DESC`？它是指下降，所以先展示最大的然后逐步减小。如果我们不写 `DESC`，那么它会假设我们想要升序排序。你也可以写`ASC`（例如让阅读代码的人更加清楚），但你不必非要这么做。

对于一些实用的数学函数，你可以查看 [`std` 中的函数](https://edgedb.com/docs/edgeql/funcops/set#function::std::sum) 以及 [`math` 模块](https://edgedb.com/docs/edgeql/funcops/math#function::math::stddev)。与其逐一查看，不如让我们做一个大查询，将其中的一些函数一并运用进来。为了使输出更友好，我们会将用于每个函数输出结果的解释字符串也编写进来，并将输出结果全部转换为 `<str>`，所以我们可以使用 `++` 将它们连接在一起。

```edgeql
WITH cities := City.population
SELECT (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>math::mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>math::stddev(cities)
);
```

这里使用了相当多的函数：

- `count()` 计算项目（item）的数量，
- `all()` 如果所有项目都匹配，则返回 `{true}`，否则返回 `{false}`，
- `sum()` 对所有项目进行相加，
- `max()` 给出数值最大的项目，
- `min()` 给出数值最小的项目，
- `math::mean()` 给出所有项目的平均值，
- `any()` 只要有一个项目匹配，则返回 `{true}`，否则返回 `{false}`，
- `math::stddev()` 给出所有项目的标准差。

输出结果也清楚地说明了它们是如何工作的：

```
{
  (
    'Number of cities: 5',
    'All cities have more than 50,000 people: false',
    'Total population: 4156229',
    'Smallest and largest population: 9100, 3500000',
    'Average population: 831245.8',
    'At least one city has more than 5 million people: false',
    'Standard deviation: 1500876.8248',
  ),
}
```

`any()`、`all()` 和 `count()` 在操作中特别有用，可以让你更了解你的数据。

## 使用 WITH 导入模块（Importing modules with WITH）

你也可以使用关键字 `WITH` 来导入模块。在上面的例子中，我们使用了 EdgeDB 的 `math` 模块中的两个函数：`math::mean()` 和 `math::stddev()`。如果我们只写 `mean()` 和 `stddev()` 会报错：

```
ERROR: InvalidReferenceError: function 'mean' does not exist
```

如果你不想每次都写模块名 `math`，你可以在 `WITH` 后面导入模块。让我们把它放到我们刚刚使用的查询中。看你是否能看出发生了什么变化：

```edgeql
WITH cities := City.population,
  MODULE math
SELECT (
  'Number of cities: ' ++ <str>count(cities),
  'All cities have more than 50,000 people: ' ++ <str>all(cities > 50000),
  'Total population: ' ++ <str>sum(cities),
  'Smallest and largest population: ' ++ <str>min(cities) ++ ', ' ++ <str>max(cities),
  'Average population: ' ++ <str>mean(cities),
  'At least one city has more than 5 million people: ' ++ <str>any(cities > 5000000),
  'Standard deviation: ' ++ <str>stddev(cities)
);
```

输出结果是一样的，但我们添加了一个 `math` 模块的导入，让我们在后面的语句中可以只写 `mean()` 和 `stddev()`。

与重命名类型相同，你还可以使用 `AS` 重命名模块（其实是为模块起个 _别名_）。所以像下面这样也可以工作：

```edgeql
WITH M AS MODULE math,
SELECT M::mean(City.population);
```

这给了我们平均值：`{831245.8}`。

## 名称的更多可计算型（Some more computables for names）

我们在本章中看到苏厄德医生（Dr. Seward）请他的老师范海辛医生 (Dr. Van Helsing) 来帮助露西（Lucy）。以下是范海辛医生在信中说他要来的开头：

```
Letter, Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc., to Dr. Seward.

“2 September.

“My good Friend,—
“When I have received your letter I am already coming to you.
```

`Abraham Van Helsing, M. D., D. Ph., D. Lit., etc., etc.` 这部分很有趣。这可能是个不错的时机让我们进一步考虑我们的 `Person` 类型中的 `name` 属性。现在我们的 `name` 属性只是一个字符串，但是我们的角色会拥有不同类别的称呼，按以下顺序：

Title（头衔） | First name（名） | Last name（姓） | Degree（学位）

所以有 'Count Dracula'（德古拉伯爵，头衔和名字），'Dr. Seward'（医生西沃德，头衔和姓名），'Dr. Abraham Van Helsing, M.D, Ph. D. Lit.'（医生亚伯拉罕·范海辛医学博士、哲学博士、文学博士等，头衔+名字+姓氏+学位），依此类推。

这会导致我们认为我们应该有类似 `first_name`、`last_name`、`title` 等称呼，然后使用可计算的公式将它们连接在一起。但话又说回来，并不是每个角色的名字都有这四个部分。像“女人 1（Woman 1）”和“旅店老板（The Innkeeper）”，我们的游戏肯定会有更多这样的命名。因此，直接去掉 `name` 或总是组合不同的部分构建名称可能不是一个好主意。但是在我们的游戏中，我们可能让角色写信或互相交谈，他们将不得不使用诸如头衔和学位之类的东西。

我们可以尝试采用中间方法。保留 `name`，并为 `Person` 添加一些属性：

```sdl
property title -> str;
property degrees -> str;
property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;
```

我们可以尝试对 `degrees` 做一些更有趣的事情，比如将 `degrees` 设置为 `array<str>`，但我们的游戏可能不需要那么高的精度，我们只是在角色间的对话中会用到学位称号。

现在是插入范海辛 (Van Helsing）的时候了：

```edgeql
INSERT NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```

现在我们可以利用这些属性在游戏中模仿一些对话了。例如：

```edgeql
WITH helsing := (SELECT NPC filter .name ILIKE '%helsing%')
SELECT (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ ',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

顺便说一下，字符串中的 `\n` 创建了一个新行，而 `\t` 用于将文本向右移动一个制表符。

我们得到结果：

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.,
        I am sorry to say that I bring bad news about Lucy.',
  ),
}
```

在有用户的标准数据库中，这要简单得多：让用户输入他们的名字、姓氏等，并使每个部分都成为一个属性。

## 其他转义字符和原始字符串（Other escape characters and raw strings）

除了 `\n` 和 `\t` 之外，还有很多其他转义字符 —— 你可以在 [此处](https://www.edgedb.com/docs/edgeql/lexical/#strings) 查看完整列表。有些很少见，但带有 `\x` 的十六进制是一个可能有用的例子。

如果你想忽略转义字符，请在引号前放置一个 `r`。让我们用上面的例子来试试，只有最后一部分有一个 `r`：

```edgeql
WITH helsing := (SELECT NPC filter .name ILIKE '%helsing%')
SELECT (
  'There goes ' ++ helsing.name ++ '.',
  'I say! Are you ' ++ helsing.conversational_name ++ '?',
  'Letter from ' ++ helsing.pen_name ++ r',\n\tI am sorry to say that I bring bad news about Lucy.'
);
```

现在我们会得到：

```
{
  (
    'There goes Abraham Van Helsing.',
    'I say! Are you Dr. Abraham Van Helsing?',
    'Letter from Abraham Van Helsing, M.D., Ph. D. Lit., etc.\n\tI am sorry to say that I bring bad news about Lucy.',
  ),
}
```

最后，如果你想保有字符串完全原始的模样，你可以在其两侧使用 `$$`。里面所有内容都将忽略所有的引号，因此你不必担心字符串会在中间断开。这是一个里面包含了一堆单引号和双引号的示例：

```edgeql
SELECT $$ "Dr. Van Helsing would like to tell "them" about "vampires" and how to "kill" them, but he'd sound crazy." $$;
```

如果没有 `$$`，看起来会是四个单独的字符串被三个未知的关键字所连接，并且会产生错误。

## 所有标量类型（All the scalar types）

你现在已经了解了所有 EdgeDB 标量类型。总结起来有：`int16`、`int32`、`int64`、`float32`、`float64`、`bigint`、`decimal`、`sequence`、`str`、`bool`、`datetime`、 `duration`、`cal::local_datetime`、`cal::local_date`、`cal::local_time`、`uuid`、`json` 和`enum`。你可以在 [此处](https://www.edgedb.com/docs/datamodel/scalars/index) 中查看它们的文档。

## 关键词 UNLESS CONFLICT ON + ELSE + UPDATE

我们在 `name` 上设置了一个 `exclusive constraint`，这样我们就不能有两个同名的角色。这个想法源于，有人可能会在书中看到一个角色并将其插入，然后其他人会尝试做同样的事情。现在我们来插入名为约翰尼（Johnny）的角色：
We put an `exclusive constraint` on `name` so that we won't be able to have two characters with the same name. The idea is that someone might see a character in the book and insert it, and then someone else would try to do the same. So this character named Johnny will work:

```edgeql
INSERT NPC {
  name := 'Johnny'
};
```

如果我们再做一次上面的插入，我们会得到这个错误：`ERROR: ConstraintViolationError: name violates exclusivity constraint`

但有时仅仅生成错误提示是不够的 —— 也许我们希望会引发其他的事情而不是直接放弃。这是 `UNLESS CONFLICT ON` 发挥作用的时候了，在它的后面会跟着 `ELSE` 来解释要做什么。通过示例可能更容易解释 `UNLESS CONFLICT ON`。这里展示了当我们要创建带有人口的 `City` 时我们可以怎么做 —— 插入，如果该城市已经存在于数据库中，则做更新。

1880 年慕尼黑（Munich）的人口为 230,023，五年后为 261,023。所以让我们假设我们正在更新 `City` 数据，且一些城市可能在数据库里已经存在，而另一些城市可能尚不存在。`INSERT` 将如下所示：

```edgeql
INSERT City {
  name := 'Munich',
  population := 261023,
} UNLESS CONFLICT ON .name
ELSE (
  UPDATE City
  SET {
    population := 261023,
  }
);
```

让我们一步一步地拆解：

首先是一个普通的插入：

```edgeql
INSERT City {
  name := 'Munich'
}
```

但是数据库里可能已经有一个同名的 `City` 了，所以它可能不起作用，于是我们添加 `UNLESS CONFLICT ON .name`。然后在它后面加上 `ELSE`，给出指示明确要做什么。

需要注意的是：这里我们不写 `ELSE INSERT`，因为我们已经在 `INSERT` 当中了。我们要为插入的类型做的是 `UPDATE`，所以是 `UPDATE City`。根本上讲，我们是从插入中取出失败的数据并更新它以重试。所以我们会这样做：

```edgeql
ELSE (
  UPDATE City
  SET {
    population := 261023
  }
);
```

这样做，我们可以保证得到一个名为慕尼黑（Munich）的 `City` 对象，且人口为 261,023，无论它是否已经存在于数据库中。

[这里是第十章中到目前为止的所有代码。](code.md)

<!-- quiz-start -->

## 章节小练习

1. 尝试通过一个插入语句插入两个 `NPC` 类型的对象，其中包含 `name`, `first_appearance` 和 `last_appearance` 信息。

   `{('Jimmy the Bartender', '1887-09-10', '1887-09-11'), ('Some friend of Jonathan Harker', '1887-07-08', '1887-07-09')}`

2. 这里还有两个要插入的 `NPC`，后面一个的最后有一个空集（因为她还没有死）。我们会遇到什么问题？

   `{('Dracula\'s Castle visitor', '1887-09-10', '1887-09-11'), ('Old lady from Bistritz', '1887-05-08', {})}`

3. 你将如何按人物姓名的最后一个字母对 `Person` 类型进行排序？

4. 尝试插入一个名为 `''` 的 `NPC`。现在，你将如何在问题 3 中执行相同的查询？

   提示：`''` 的长度为 0，这可能会有问题。

5. 范海辛医生（Dr. Van Helsing ）有一份 `MinorVampire` 的名单，上面有他们的名字和力量。我们的数据库中已经有一些 `MinorVampire` 了。如果对象已经存在，你将如何在确保 `UPDATE` 的同时 `INSERT` 不存在的？ 


[可以在这里查看答案。](answers.md)

<!-- quiz-end -->

__接下来：__ _他们会相信范海辛所谓的真相吗？_
