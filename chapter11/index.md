---
tags: Writing Functions, Multiplication
---

# 第十一章 - 露西怎么了？

> 范海辛医生（Dr. Van Helsing）认为露西（Lucy）正在被吸血鬼纠缠。他还没有告诉其他人，因为他们不会相信他，他告诉大家应该关上窗户，并在各个地方都放上大蒜。大家很困惑，但苏厄德医生（Dr. Seward）让他们听从，因为范海辛医生是他认识的最聪明的人。见效了！露西好了起来。但一天晚上，露西的母亲走进房间，认为屋里闻起来很糟糕！于是她打开了窗户为了呼吸新鲜的空气。第二天，露西醒来，脸色苍白，再次病重。每次有人犯这样的错误，德古拉（Dracula）都会进她的房间，并且每次病重都需要男人们献血来帮露西好起来。与此同时，伦菲尔德（Renfield）继续尝试吃活物，苏厄德医生（Dr. Seward）无法理解他。后来有一天他不想多说话，只是一直在说：“我不想和你说话：你现在不算数；大师就在身边。” 

我们开始在书中看到越来越多的多个角色都参与的事件。有些事件是三个男人和范海辛博士（Dr. Van Helsing）在一起，有些事件只有露西（Lucy）和德古拉（Dracula）。之前还有事件是乔纳森·哈克（Jonathan Harker）和德古拉（Dracula）在一起，还有乔纳森·哈克和三个女吸血鬼，等等。在我们的游戏中，我们可以使用 `Event` 类型将所有内容组合在一起，如：人物、时间、地点等等。

这个 `Event` 类型的定义有点长，但它会是我们游戏中事件的主要类型，所以需要详细说明。我们可以这样组合：

```sdl
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  property exact_location -> tuple<float64, float64>;
  property east -> bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ 'E' if .east = true else 'W';
}
```

你可以看到大多数属性都有 `required`，因为如果 `Event` 类型没有我们需要的所有信息，它就没有什么用处了。它总是需要描述、时间、地点和参与人员。有趣的部分是 `url` 属性：它是一个可计算的，如果我们需要，它可以为我们提供地图中指向该位置的确切 url。

我们生成的 url 需要知道这个位置是在格林威治（Greenwich）的东部还是西部，以及它们是北部还是南部。如下是 Bistritz 的 url：

`https://geohack.toolforge.org/geohack.php?pagename=Bistri%C8%9Ba&params=47_8_N_24_30_E`

对我们来说幸运的是，书中的事件都发生在地球的北部。所以 `N` 总是会在那里。但有时他们在格林威治东部，有时在西部。为了说明是东方还是西方，我们可以使用一个简单的 `bool`。然后在 `url` 属性中，我们将所有相关属性放在一起以创建链接，如果 `east` 为 `true`，则以 `'E'` 结束，否则以 `'W'` 结束。

（当然，如果我们接收的经度是简单的正负数（+ 表示东，- 表示西），那么 `east` 可以是一个可计算的表达：`property Eastern := true if exact_location.0 > 0 else false`。但是对于这个架构，我们会假设我们从某个地方以这种格式获取数字：`[50.6, 70.1, true]`）

让我们插入本章中的一个事件。它发生在 9 月 11 日晚上，当时范海辛医生（Dr. Van Helsing）正试图帮助露西（Lucy）。你可以看到 `description` 属性只是我们编写的一个字符串，以便于稍后进行搜索。它可长可短，这取决于你，我们甚至可以把书中的某些部分粘贴进去。

```edgeql
INSERT Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1887, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1887, 9, 11, 23, 0, 0),
  place := (SELECT Place FILTER .name = 'Whitby'),
  people := (SELECT Person FILTER .name ILIKE {'%helsing%', '%westenra%', '%seward%'}),
  exact_location := (54.4858, 0.6206),
  east := false
};
```

有了所有这些信息，我们现在可以通过描述、角色、位置等来查询事件。

现在让我们查询所有包含 `garlic flowers` 一词的事件：

```edgeql
SELECT Event {
  description,
  start_time,
  end_time,
  place: {
    __type__: {
      name
    },
    name
  },
  people: {
    name
  },
  exact_location,
  url
} FILTER .description ILIKE '%garlic flowers%';
```

这生成了一个不错的输出，向我们展示了结果事件的一切： 

```
{
  Object {
    description: 'Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.',
    start_time: <cal::local_datetime>'1857-09-11T18:00:00',
    end_time: <cal::local_datetime>'1857-09-11T23:00:00',
    place: {Object {__type__: Object {name: 'default::City'}, name: 'Whitby'}},
    people: {
      Object {name: 'John Seward'},
      Object {name: 'Lucy Westenra'},
      Object {name: 'Abraham Van Helsing'},
    },
    exact_location: (54.4858, 0.6206),
    url: 'https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W',
  },
}
```

url 也是正确的。它是：<https://geohack.toolforge.org/geohack.php?params=54.4858_N_0.6206_W> 点击它可以直接查看到惠特比市。

## 自定义函数（Writing our own functions）

基于之前的信息，我们可以看到伦菲尔德（Renfield）非常强壮：他的力量值是 10，而乔纳森（Jonathan）的力量是 5。

我们现在可以用它来实践如何制作函数。由于 EdgeQL 是强类型的，你必须在签名中同时指明输入类型和返回类型。例如，输入 int16 类型并返回 float64 类型的函数签名如下所示：

```sdl
function does_something(input: int16) -> float64
```

`->` 细箭头用于显示返回值。

对于函数体，我们执行以下操作：

- 写下 `using` 然后跟上一个 `()` 括号，
- 在括号里写下函数定义，
- 用分号结束定义（在 `)` 后面写分号）。 

下面这个是一个非常简单的函数，它接受一个数字并最终返回一个字符串：

```sdl
function make_string(input: int64) -> str
  using (<str>input);
```

就是这样！

现在让我们编写一个函数，让两个角色进行战斗。我们将使逻辑尽可能简单，即具有更多力量的角色获胜，如果他们的力量相同，则第二个玩家获胜。

```sdl
function fight(one: Person, two: Person) -> str
  using (
    SELECT one.name ++ ' wins!' IF one.strength > two.strength ELSE two.name ++ ' wins!'
  );
```

到目前为止，只有乔纳森（Jonathan）和伦菲尔德（Renfield）拥有 `strength` 属性，我们让他们较量一番：

```edgeql
WITH
  renfield := (SELECT Person filter .name = 'Renfield'),
  jonathan := (SELECT Person filter .name = 'Jonathan Harker')
SELECT (
  fight(jonathan, renfield)
);
```

结果符合我们的预期：`{'Renfield wins!'}`

当我们通过过滤器选择人物时，我们最好加上 `LIMIT 1`。因为 EdgeDB 返回的是集合，如果它得到多个结果，那么它将针对每个可能的组合对每个结果使用该函数。EdgeDB 处理这个的方式是通过笛卡尔乘法（Cartesian multiplication），现在让我们进一步了解一下。

## 笛卡尔乘法（Cartesian multiplication）

笛卡尔乘法（Cartesian multiplication）听起来很吓人，但实际上只是意味着“将一个集合中的每个项目分别连接到另一个集合中的每个项目”。通过图示会更加容易理解，幸运的是维基百科已经为我们制作了插图。当你在 EdgeDB 中将集合相乘时，你会得到笛卡尔乘积，如下所示：

![](cartesian_product.svg)

来源：[维基百科的用户“quartl”](https://en.wikipedia.org/wiki/Cartesian_product#/media/File:Cartesian_Product_qtl1.svg)

意味着如果我们为我们的 `fight()` 函数对 `Person` 执行 `SELECT`，它将按照以下公式运行函数：

- `{the number of items in the first set}` \* `{the number of items in the second set}`

因此，如果第一个集合中有两个，第二个结合中有三个，该函数将被运行 6 次。

为了演示，让我们给函数的两个输入里都放置三个对象。同时使用 `++` 在输出结果里添加更清晰的文字说明：

```edgeql
WITH
  first_group := (SELECT Person FILTER .name in {'Jonathan Harker', 'Count Dracula', 'Arthur Holmwood'}),
  second_group := (SELECT Person FILTER .name in {'Renfield', 'Mina Murray', 'The innkeeper'}),
SELECT (
  first_group.name ++ ' fights against ' ++ second_group.name ++ '. ' ++ fight(first_group, second_group)
);
```

这是输出。总共有九场战斗，第 1 组中的每个人与第 2 组中的每个人分别战斗一次。

```
{
  'Count Dracula fights against The innkeeper. The innkeeper wins!',
  'Count Dracula fights against Mina Murray. Mina Murray wins!',
  'Count Dracula fights against Renfield. Renfield wins!',
  'Jonathan Harker fights against The innkeeper. The innkeeper wins!',
  'Jonathan Harker fights against Mina Murray. Mina Murray wins!',
  'Jonathan Harker fights against Renfield. Renfield wins!',
  'Arthur Holmwood fights against The innkeeper. The innkeeper wins!',
  'Arthur Holmwood fights against Mina Murray. Mina Murray wins!',
  'Arthur Holmwood fights against Renfield. Renfield wins!',
}
```

如果你去掉过滤器，只用 `SELECT Person`，你会得到超过 100 个结果。EdgeDB 默认只显示前 100 个，在显示 100 个结果后会显示：

`` ... (further results hidden `\set limit 100`)``

[这里是第十一章中到目前为止的所有代码。](code.md)

<!-- quiz-start -->

## 章节小练习

1. 如何编写一个名为 `lucy()` 的函数，它将只返回与名称“Lucy Westenra”匹配的所有 `NPC` 类型？

2. 如何编写一个函数，其接受两个字符串并会返回名称与输入的两个字符串任意一个匹配的所有 Person 对象？

   提示：尝试使用 `SET OF Person` 作为返回类型。

3. 下面的语句将会输出什么？

   ```edgeql
   SELECT {'Jonathan', 'Arthur'} ++ {' loves '} ++ {'Mina', 'Lucy'} ++ {' but '} ++ {'Dracula', 'The inkeeper'} ++ {' doesn\'t love '} ++ {'Mina', 'Jonathan'};
   ```

4. 如何制作一个函数来计算一个城市比另一个城市大多少倍？

5. `SELECT (City.population + City.population)` 和 `SELECT ((SELECT City.population) + (SELECT City.population))` 会产生不同的结果吗？

[可以在这里查看答案。](answers.md)

<!-- quiz-end -->

__接下来：__ _一天晚上，露西：“是什么在拍打窗户？听起来像是蝙蝠什么的……"_
