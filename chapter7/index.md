---
tags: Constraint Delegation, $ Parameters
---

# 第七章 - 乔纳森最终“离开”了城堡

> 乔纳森（Jonathan）白天偷偷溜进德古拉（Dracula）的房间，看到他睡在棺材里。现在他知道了德古拉是一个吸血鬼。几天后，德古拉伯爵说他明天就要走了。乔纳森认为这是一个好机会，并要求现在就离开。 德古拉打开门并回应道：“好吧，如你愿意……” 但是外面有很多狼，它们嚎叫着，发出很大的声音。德古拉又道：“你可以离开了，再见！” 乔森纳知道，如果他走出去，狼群将会杀掉他。他也知道是德古拉召唤的狼群，于是请他把门关上。德古拉微笑着关上了门……他知道乔纳森被困住了。之后，乔纳森听到德古拉告诉三个女吸血鬼，她们可以在他明天离开后享用他。第二天，德古拉的朋友把他带走了（他在棺材里），乔纳森独自一人留下……不久，夜幕降临。所有的门都锁上了。他决定从窗户爬出去，因为他宁愿摔死，也不愿意与女吸血鬼们单独在一起。他在日记中写道“再见了，各位！米娜！” ，然后开始爬墙。

## 更多约束（More constraints）

在乔纳森爬墙时，我们可以继续处理我们的数据库架构（schema）。在我们的书中，没有相同名字的角色，所以应该只有一个米娜·默里（Mina Murray），一个德古拉伯爵等等。这是在 `Person` 类型的 `name` 上放置 [constraint（约束）](https://edgedb.com/docs/datamodel/constraints#ref-datamodel-constraints)的好时机，以确保我们不会有重复的插入。`constraint` 是一个限制，我们人类的在 `age` 中已经看到限制，即只能达到 120 岁。对于 `name`，我们可以给它增加一个名为 `constraint exclusive` 的限制，以防止两个相同类型的对象具有相同的名称。您可以在属性后的块中放置一个 `constraint`，如下所示：

```sdl
abstract type Person {
  required property name -> str { ## Add a block
      constraint exclusive;       ## and the constraint
  }
  multi link places_visited -> Place;
  link lover -> Person;
}
```

现在我们知道了将只有一个 `Jonathan Harker`，一个 `Mina Murray` 等等。在显示生活中，这对那些类似邮箱地址、用户 ID 等我们希望具有唯一性的属性十分有用。在我们的数据库里，我们也将给 `Place` 里的 `name` 添加 `constraint exclusive`，因为这些地方的名字也是唯一的：

```sdl
abstract type Place {
  required property name -> str {
      constraint exclusive;
  };
  property modern_name -> str;
  property important_places -> array<str>;
}
```

## 传递约束（Passing constraints with delegated）

现在，我们的 `Person` 类型的属性 `name` 具有 `constraint exclusive`，任何扩展自 `Person` 的类型都不可以拥有同样的名称。这对于本教程中的游戏来说很好了，因为我们已经知道书中的所有角色名称，并且不打算制作任何真正的 `PC` 类型的对象。但是如果我们稍后想创建一个名为 Jonathan Harker 的 `PC` 该怎么办？现在是不允许的，因为我们已经有了一个同名的 `NPC`，`NPC` 的 `name` 来自 `Person` 类型。

幸运的是这里有一个简单的办法绕过它：在 `constraint` 前面添加关键词 `delegated`。这将约束“委托（delegates）”（传递）给子类型，因此排他性检查将分别针对 `PC`、`NPC`、`Vampire` 等进行（而不在他们彼此之间进行检查）。即你只需要额外加上关键字 `delegated` 在之前的列子当中：

```sdl
abstract type Person {
  required property name -> str {
    delegated constraint exclusive;
  }
  multi link places_visited -> Place;
  link lover -> Person;
  property strength -> int16;
}
```

有了它，你可以拥有最多一个叫 Jonathan Harker 的 `PC`对象、最多一个叫 Jonathan Harker 的 `NPC` 对象、最多一个叫 Jonathan Harker 的 `Vampire` 对象以及最多一个叫 Jonathan Harker 的任何扩展自 `Person` 类型的对象。

## 在查询中使用函数（Using functions in queries）

让我们也考虑一下我们的游戏机制。书里说城堡里的门对于乔纳森来说太难打开了，但是德古拉足够强壮可以打开所有。在真正的游戏中，它会更复杂，但我们可以尝试一些简单的方法来模仿这个事实：

- 门有力量（strength），人也有力量。
- 如果人的力量大于门，则他/她可以打开门。

因此我们将创建一个 `Castle` 类型，并给它一些门（即设置属性 `doors`）。现在我们想给这些门设置一些表示“强度（strength）”的数字，所以我们将 `doors` 设为一个 `array<int16>`：

```sdl
type Castle extending Place {
    property doors -> array<int16>;
}
```

然后我们假设这里有三个主要的门用来出入德古拉城堡，所以我们按如下方式 `INSERT` 它们。

```edgeql
INSERT Castle {
  name := 'Castle Dracula',
  doors := [6, 19, 10],
};
```

然后我们也将添加一个 `property strength -> int16;` 到 `Person` 类型。这个属性将不是必需的，因为我们并不知道本书中所有人的力量……尽管如果游戏需要我们可以在之后将其设置为 `required`。

现在我们给乔纳森（Jonathan）一个等于 5 的力量值。像之前一样，我们很容易使用 `UPDATE` and `SET` 进行更新：

```edgeql
UPDATE Person FILTER .name = 'Jonathan Harker'
SET {
  strength := 5
};
```

好的。我们知道乔纳森无法冲出城堡，但让我们尝试使用一个查询语句来展示这个事实。要做到冲出城堡，他需要拥有比门还大的力量。或者换句话说，他需要比最弱的门拥有更大的力量。

幸运的是，有一个叫做 `min()` 的函数可以给出一个集合中的最小值，所以我们可以利用它。如果乔纳森的力量大于拥有最小数值的门的力量，他则可以逃脱。下面的查询看起来应该可以工作，但并不完全是：

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  castle_doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT jonathan_strength > min(castle_doors);
```

这里会报错：

```
error: operator '>' cannot be applied to operands of type 'std::int16' and 'array<std::int16>'
```

我们可以 [查看这个函数签名](https://edgedb.com/docs/edgeql/funcops/set#function::std::min) 来发现问题：

```
std::min(values: SET OF anytype) -> OPTIONAL anytype
```

重要的部分是 `SET OF`：它需要的是一个集合，所以我们用大括号括起来。但我们不能只在数组前后放置大括号，因为这样它就会变成一个项目（一个数组）的集合。所以 `SELECT min({[5, 6]});` 只返回 `{[5, 6]}`，而不是 `{5}`，因为 `{[5, 6]}` 里只有一个数组，所以 `{}` 里最小的数组只能是 `[5, 6]`。这也意味着 `SELECT min({[5, 6], [2, 4]});` 将会返回 `{[2, 4]}`（而不是 2）。这不是我们想要的。

因此，我们实际想要使用的是 [array_unpack()](https://edgedb.com/docs/edgeql/funcops/array#function::std::array_unpack) 函数，它接受一个数组并可以将其解包为一个集合。所以我们将对 `weakest_door` 使用该函数：

```edgeql
WITH
  jonathan_strength := (SELECT Person FILTER .name = 'Jonathan Harker').strength,
  doors := (SELECT Castle FILTER .name = 'Castle Dracula').doors,
SELECT jonathan_strength > min(array_unpack(doors));
```

我们将得到 `{false}`。完美！现在我们成功展示了乔纳森不能打开任何门。他将不得不从窗户爬出逃跑。

除了 `min()`，当然还有 `max()`。`len()` 和 `count()` 也都很有用：`len()` 可以给出一个对象的长度，`count()` 可以给出它们的数量。下面是一个使用 `len()` 获取所有 `NPC` 类型对象的名称长度的示例：

```edgeql
SELECT (NPC.name, 'Name length is: ' ++ <str>len(NPC.name));
```

别忘了我们需要做一个 `<str>` 的强制转换，因为 `len()` 返回的是一个整数，且 EdgeDB 无法连接一个字符串和一个整数。打印结果如下：

```
{
  ('The innkeeper', 'Name length is: 13'),
  ('Mina Murray', 'Name length is: 11'),
  ('Jonathan Harker', 'Name length is: 15'),
}
```

另一个使用 `count()` 的例子也需要做 `<str>` 的强制转换：

```edgeql
SELECT 'There are ' ++ <str>(SELECT count(Place) - count(Castle)) ++ ' more places than castles';
```

打印结果是：`{'There are 6 more places than castles'}`.

在之后的几章中，我们将学习如何创建自己的函数来缩短查询时间。

## 使用 $ 设置参数（Using $ to set parameters）

假设我们需要一直查找 `City` 类型，使用这种查询：

```edgeql
SELECT City {
  name,
  population
} FILTER .name ILIKE '%a%' AND len(.name) > 5 AND .population > 6000;
```

这将正常工作并返回一个城市：`{Object {name: 'Buda-Pesth', population: 402706}}`。

但是对最后一行包含的所有过滤器的更改可能有点烦人：在我们再次按 Enter 执行语句之前，会因为删除和重新输入需要很多的光标移动。

这正是使用 `$` 向查询添加参数的好时机。我们可以给参数一个名字，EdgeDB 会在每个查询中询问我们给它什么值。让我们从一些非常简单的事情开始：

```edgeql
SELECT City {
  name
} FILTER .name = 'London';
```

现在让我们把 `'London'` 改为 `$name`。注意：这照样不会工作，猜猜为什么？

```edgeql
SELECT City {
  name
} FILTER .name = $name;
```

这问题在于 `$name` 可以是任何，EdgeDB 不知道它将是什么类型。错误提示同样告诉我：`error: missing a type cast before the parameter`。所以因为它是一个字符串，我们将使用 `<str>` 进行转换：

```edgeql
SELECT City {
  name
} FILTER .name = <str>$name;
```

当我们这样做时，我们会收到一个提示，要求我们输入值：`Parameter <str>$name:`。输入 London，不需要引号，因为 EdgeDB 已经知道它是一个字符串了。这结果是：`{Object {name: 'London'}}`。

现在让我们使用三个参数来创建一个更复杂（和有用）的查询。我们将它们称为 `$name`、`$population` 和 `$length`。不要忘记对它们进行类型转换：

```edgeql
SELECT City {
  name,
  population,
} FILTER
    .name ILIKE '%' ++ <str>$name ++ '%'
  AND
    .population > <int64>$population
  AND
    <int64>len(.name) > <int64>$length;
```

由于有三个参数，EdgeDB 会要求我们输入三个值。下面是它的一个示例：

```
Parameter <str>$name: u
Parameter <int64>$population: 2000
Parameter <int64>$length: 5
```

因此，这将给出所有名称中包含 u、人口超过 2000 且名称长度超过 5 个字符的 `City` 类型的对象。结果是：

```
{
  Object {name: 'Buda-Pesth', population: 402706},
  Object {name: 'Munich', population: 230023},
}
```

参数在插入语句中也同样有效。这是一个 `Time` 插入，提示用户输入小时、分钟和秒：

```edgeql
SELECT(
  INSERT Time {
    date := <str>$hour ++ <str>$minute ++ <str>$second
  }
) {
  date,
  local_time,
  hour,
  awake
};
Parameter <str>$hour: 10
Parameter <str>$minute: 09
Parameter <str>$second: 09
```

输出结果是：

```
{
  default::Time {
    date: '100909',
    local_time: <cal::local_time>'10:09:09',
    hour: '10',
    awake: 'asleep',
  },
}
```

请注意，强制转换意味着你只能输入 `10`，而不是 `'10'`。

那么，如果你只想拥有一个 _可选的_ 参数呢？没问题，只需将 `OPTIONAL` 放在类型名称之前（放在 `<>` 括号内）。因此，如果你希望所有内容都是可选的，则上面的插入内容将如下所示：

```edgeql
SELECT(
  INSERT Time {
    date := <OPTIONAL str>$hour ++ <OPTIONAL str>$minute ++ <OPTIONAL str>$second
  }
) {
  date,
  local_time,
  hour,
  awake
};
```

当然，`Time` 类型的 `date` 属性需要正确的格式，所以上面的做法不是一个好主意。这里只是为了展示一下你可以如何做。

`OPTIONAL` 的相反是 `REQUIRED`，但因为它是默认的，所以你不需要总是写上它。

我们在之前章节里学到的关键字 `UPDATE` 也可以使用参数，所以总共有四个关键字我们可以使用参数：`SELECT`、`INSERT`、`UPDATE` 和 `DELETE`。

[→ 点击这里查看第 7 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何选择出每一个 City 及他们名字的长度？

2. 如何选择出每一个 City 并展示其 `name` 长度减去 `modern_name` 长度的结果，如果 `modern_name` 不存在，则显示 0。 

3. 如果在上一题中想用 `'Modern name does not exist'` 替代 0，作为 `modern_name` 不存在时的结果显示，该如何做？

4. 如果已经有 7 个 NPC 存在，你将如何插入名为“NPC number 8”的 NPC？

5. 如何选择出名字最短的 `Person` 类型对象？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _瓦尔纳市的工人将箱子装上了船，德古拉就在其中之一……_
