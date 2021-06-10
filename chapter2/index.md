---
tags: Scalar Types, Abstract Types, Filter
leadImage: illustration_02.jpg
---

# 第二章 - 在比斯特里茨的酒店

我们继续阅读这个故事，并思考哪些信息我们需要存入数据库。重要的信息以粗体显示：

> 乔纳森·哈克（Jonathan Harker）在 **比斯特里茨（Bistritz）** 发现了一家酒店, 叫做 **金克朗酒店（Golden Krone Hotel）**。他在酒店里收到一封来自德拉库拉的欢迎信，信中说明德拉库拉正在 **城堡（castle）** 里等他。乔纳森·哈克（Jonathan Harker）明天必须搭乘 **马车（horse-driven carriage）** 才能到达那里。我们也看到乔纳森·哈克（Jonathan Harker）来自 **伦敦（London）**。金克朗酒店（Golden Krone Hotel）的老板似乎很害怕德古拉。他不想让乔纳森（Jonathan）离开并表明前往城堡会很危险，但乔纳森（Jonathan）并没有听进去。一位老太太给了乔纳森（Jonathan）一个金色的十字架，并说这会保护他。乔纳森（Jonathan）感到尴尬，但认为这可能是出于礼貌，他并不知道之后这会对他有多大的帮助。

现在我们开始看一下关于这座城市的一些细节。通过阅读这个故事，我们看到我们可以添加另一个属性给 `City`，我们叫它 `important_places`。比如可以去像 **金克朗酒店（Golden Krone Hotel）** 这样的地方。我们尚不确定这些地方是否将拥有属于他们自己的类型，因此我们只是定义它为一个字符串数组，像这样：`property important_places -> array<str>;` 我们可以把这些重要地点的名字放进去，也许之后还会发展出更多的内容。现在它看起来像这样：

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
```

现在我们对 Bistritz 的原始插入将如下所示：

```edgeql
INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

## 枚举、标量类型和扩展（Enums, scalar types, and extending）

到目前位置，我们在书中有提到两种交通工具：火车和马车。这本书以 1887 年为背景，我们的游戏将让角色使用当年可用的交通工具。这里的 `enum`（枚举）可能是最好的选择，因为 `enum` 是在选项之间做出一个选择。枚举的变量应该用大写驼峰式（UpperCamelCase）进行书写。

这里我们第一次看到单词 `scalar`：这是一个“标量类型”（`scalar type`），因为它一次只保存一个值。其他类型（`City`, `Person`）是“对象类型”（`object types`）因为他们能够同时保存多个值。

另一个我们第一次看到的关键词是 `extending`：它意味着以一个类型作为基础并扩展它。这使你对你想要扩展的类型有更多的权利，可以添加更多选项。我们将按如下来写 `Transport` 类型：

```sdl
scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;
```

你是否留意到 `scalar type` 是以一个分号结尾的，而其他类型并非如此？这是因为其他类型有一个 `{}` 以构成一个完整的表达式。但是这里的单行代码我们并没有 `{}`，所以在这里我们需要用分号来说明表达式的结束。

这个 `Transport` 类型将被用于我们游戏中的玩家角色，而不是书中的人物（他们的故事和选择已成定局）。这意味着我们需要一个 `PC` 类型和一个 `NPC` 类型，但我们的 `Person` 类型也应该保留 - 我们可以将它用作两者的基本类型。为此，我们可以让 `Person` 成为一个 `abstract type` 而不仅仅是一个 `type`。然后有了这个抽象类型，我们可以对其他 `PC` 和 `NPC` 类型使用关键字 `extending`。

所以现在这部分架构看起来像这样：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
}

type PC extending Person {
  required property transport -> Transport;
}

type NPC extending Person {
}
```

现在书中的角色将是 `NPC`（非玩家角色），而 `PC` 是在考虑我们的游戏的情况下设定的。因为 `Person` 现在是一个抽象类型，我们不能再对其进行直接的插入。如果你尝试执行 `INSERT Person {name := 'Mr. HasAName'};`，将会收到错误提示：

```
error: cannot insert into abstract object type 'default::Person'
  ┌─ query:1:8
  │
1 │ INSERT Person {
  │        ^^^^^^^ error
```

没关系 —— 只要将 `Person` 改为 `NPC`，它就可以工作了。

此外，`SELECT` 一个抽象类型是没有问题的 —— 它将会选择出所有从它扩展出的类型。

让我们也试验一下玩家角色。我们创建一个名叫 Emil Sinclair 的人，他开始乘坐马车旅行。我们也将 `City` 给他，于是他也拥有了三个造访过的城市。

```edgeql
INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := <Transport>'HorseDrawnCarriage',
};
```

`places_visited := City` 是对 `places_visited := (SELECT City)` 的简写 —— 你不是必须每次都输入 `SELECT` 部分。

请注意，我们并没有只是写了 `HorseDrawnCarriage`，我们必须选择枚举类型 `Transport` 并选择其中一个枚举值。`<>` 尖括号做 _casting_，意思是将一种类型转换为另一种类型。EdgeDB 不会自行尝试将一种类型更改为另一种类型，除非你要求它进行转换。这就是为什么下面的语句不会给我们 `true` 的原因：

```edgeql
SELECT 'Feet' IS Transport;
```

我们将得到一个 `{false}` 的输出结果，因为 'Feet' 仅仅是一个 `str`。但是下面将会返回 `true`：

```edgeql
SELECT <Transport>'Feet' IS Transport;
```

然后我们如愿得到 `{true}`。

如果需要，你可以一次性转换多次。下面这个例子不是你需要做的，只是为了展示如果你愿意，你可以如何一遍又一遍地做类型转换：

```edgeql
SELECT <str><int64><str><int32>50 is str;
```

这个也会返回我们 `{true}`，因为我们所做的只是询问它是否是一个 `str`，且它确实是。

类型转换从右往左执行，最后的转换是在最左侧。因此，`<str><int64><str><int32>50` 意味着“50 先变成了 int32，再变成了 str，又变成了 int64，最后又变成了 str。

此外，需要注意类型转换仅适用于标量类型 `scalar type`：用户创建的对象类型，如 `City` 和 `Person` 都太复杂，并不能简单地相互转换。

## 过滤（Filter）

最后，在我们结束第二章内前，让我们一起来学习一下如何使用 `FILTER`。你可以在 `SELECT` 中的花括号后面使用 `FILTER` 来控制只显示某些结果。让我们用 `FILTER` 来仅显示名为“Emil Sinclair”的 `Person` 类型：

```edgeql
SELECT Person {
  name,
  places_visited: {name},
} FILTER .name = 'Emil Sinclair';
```

`FILTER .name` 是 `FILTER Person.name` 的缩写。如果你愿意，你也可以写 `FILTER Person.name` —— 它们是一样的。

输出结果如下：

```
{Object {name: 'Emil Sinclair', places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}}}}
```

现在让我们来过滤城市。一种灵活的搜索方式是使用“LIKE”或“ILIKE”来匹配字符串的一部分。

- `LIKE` 是区分大小写的：“Bistritz”可以匹配“Bistritz”，但和“bistritz”并不匹配.
- `ILIKE` 是不区分大小写的（ILIKE 中的 I 是指**不敏感（insensitive）**），所以“Bistritz”可以匹配“BiStRitz”，也可以匹配“bisTRITz”。

你也可以通过添加 `%`在你想匹配部分的左侧或右侧以示意匹配规则。以下是匹配**粗体**部分的一些示例：

- `LIKE Bistr%` 可以匹配到 “**Bistr**itz”（但不匹配 “bistritz”），
- `ILIKE '%IsTRiT%'` 可以匹配到 “B**istrit**z”，
- `LIKE %athan Harker` 可以匹配到 “Jon**athan Harker**”，
- `ILIKE %n h%` 可以匹配到 “Jonatha**n H**arker”。

让我们用 `FILTER` 过滤出所有首字母是大写字母 B 的城市。这意味着我们需要使用 `LIKE`，因为它是对大小写敏感的：

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name LIKE 'B%';
```

这是输出结果：

```
  Object {name: 'Buda-Pesth', modern_name: 'Budapest'},
  Object {name: 'Bistritz', modern_name: 'Bistrița'},
```

你也可以用 `[]` 方括号索引一个字符串，从 0 开始。比如，字符串“Jonathan”的索引如下所示：

```
J o n a t h a n
0 1 2 3 4 5 6 7
```

因此 `'Jonathan'[0]` 是“J”，`'Jonathan'[4]` 是“t”。

让我们试一下下面的语句：

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name[0]; = 'B'; # First character must be 'B'
```

这同样会给出我们想要的结果。不过要小心：如果你将数字设置得太高（超过字符串本身的长度），那么它会尝试在字符串之外进行搜索，这会带来错误。比如，如果我们将 0 更改为 18 (`FILTER .name[18]; = 'B';`)，我们将得到：

```
ERROR: InvalidValueError: string index 18 is out of bounds
```

此外，如果你有一个名字为 `''` 的 `City` 类型，即使搜索索引为 0 也会导致错误。

你还可以切片一个字符串以得到字符串的一部分。因为“Jonathan”从 0 开始，它的索引值如下所示：

```
|J|o|n|a|t|h|a|n|
0 1 2 3 4 5 6 7 8
```

它的长度是 8 个字符，因此它完全介于 0 和 8 之间。如果你在索引 2 和 5 之间“slice”它，你会得到“nat”（`'Jonathan'[2:5]` = 'nat'），因为它开始于 2，直到 5 —— 但并不包括索引 5 对应的字符。

负的索引值从“Jonathan”的末尾开始计数，即 8，所以 -1 对应的是 `8 - 1` (= 7)，以此类推。

那么，如果你想确保不会因索引号数字过高而引发错误该怎么办？在这里，您可以使用带有空参数的 `LIKE` 或 `ILIKE`，因为它只会给出一个空集：`{}` 而不是错误。如果属性中有可能包含太短的数据，`LIKE` 和 `ILIKE` 比使用索引更保险。这里需要强调：

- 在 Edgedb 中，“无数据”会被显示为空集：`{}`；
- `""`（一个空字符串）实际上也是数据。

记住它们有助于你理解他们两者之间的行为。

最后，你是否注意到我们刚刚用 `#` 写了一个注释？EdgeDB 中的注释很简单：一行中 `#` 右侧的任何内容都会被忽略，被视为“注释”。

因此语句：

```edgeql
SELECT 1887#0503 is the first day of the book Dracula when...
;
```

只是会返回 `{1887}`.

[这里是第二章中到目前为止的所有代码。](code.md)

<!-- quiz-start -->

## 章节小练习

1. 使用类型转换修改语句 `SELECT '99' + '1'`，使其输出结果为 `{100}`；
2. 选择出所有以“Mu”开头的 `City` 类型（需要区分大小写）；
3. 选择出每个 `NPC` 名字的第三个字母（即索引号是 2）；
4. 假设有一个抽象类型叫做 `HasAString`:

   ```sdl
   abstract type HasAString {
     property string -> str
   };
   ```

   你将如何修改 `Person` 类型成为 `HasAString` 的扩展类型？

5. 下面的查询仅会展示造访过的地方的 id。请问如何展示它们的名字？

   ```edgeql
   SELECT Person {
     places_visited
   };
   ```

[可以在这里查看答案。](answers.md)

<!-- quiz-end -->

__下一章__ _乔纳森坐上马车，前往寒冷的山脉。_
