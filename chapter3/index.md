---
tags: Constraints, Deleting
leadImage: illustration_03.jpg
---

# 第三章 - 乔纳森前往德古拉城堡

在本章中，我们将开始考虑时间，正如你从乔纳森·哈克（Jonathan Harker）所做的事情中可以看到的：

> 乔纳森·哈克（Jonathan Harker）乘坐马车穿越群山，刚刚抵达德古拉城堡。旅途很糟糕：到处都是雪、奇怪的蓝色火焰和狼。他抵达时，已经是晚上了。他见到了德古拉伯爵，他们聊了一夜。 不过，德古拉在太阳升起之前就离开了，因为吸血鬼会被阳光伤害。日子一天天过去，乔纳森仍然不知道他是吸血鬼。但他确实注意到了一些奇怪的事情：城堡似乎完全空无一人。如果德古拉这么富有，那么他的仆人都在哪里？桌子上的饭菜是谁做的？但乔纳森发现德古拉的历史故事非常有趣，所以到目前为止他仍然很享受他的这次旅行。

现在我们完全来到了德古拉的城堡，所以是时候创建一个 `Vampire` 类型了。我们可以基于 `abstract type Person` 扩展出它，因为该类型有 `name` 和 `places_visited`，它们也同样适用于 `Vampire`。但是吸血鬼与人类不同，它们可以永生。我们可以添加 `age` 到 `Person`，以便所有其他类型也可以使用它。那么 `Person` 看起来像这样：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  property age -> int16;
}
```

`int16` 意味着 16 位（2 字节）整数，它有足够的空间用于 -32768 到 +32767。对于年龄来说，这绰绰有余，所以我们不需要用更大的 `int32` 或 `int64` 类型。我们也不希望它成为“必需属性” `required property`，因为我们并不关心每一个人的年龄。

但是我们不会期待 `PC` 和 `NPC` 活到 32767 岁，所以我们现在只给 `Vampire` 提供 `age`，之后我们再考虑在其他类型上用 `age`。我们将基于 `Person` 扩展出 `Vampire` 类型，并添加 `age`：

```sdl
type Vampire extending Person {
  property age -> int16;
}
```

现在我们创建德古拉伯爵（Count Dracula）。我们知道他住在罗马尼亚（Romania），但这不是一个城市。因此是时候调整一下 `City`（城市）类型了。我们将其更名为 `Place`（地点），并使其成为 `abstract type`（抽象类型），然后基于其扩展出 `City`。我们还将添加一个有类似作用的 `Country` 类型。现在它们看起来像这样：

```sdl
abstract type Place {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}

type City extending Place;

type Country extending Place;
```

我们需要将 `Person` 类型里的 `places_visited` 由 `City` 修改为 `Place`。毕竟，角色们不仅可以访问城市，还可以访问更多地方：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> Place;
}
```

现在很容易创建一个 `Country`，只需插入并给它一个名字。我们马上快速插入匈牙利（Hungary）和罗马尼亚（Romania）两个 `Country` 类型的对象：

```edgeql
INSERT Country {
  name := 'Hungary'
};
INSERT Country {
  name := 'Romania'
};
```

（顺便说一下，你可能注意到了 `important_places` 仍然是一个字符串数组 `array<str>`，且改为 `multi link` 可能会更好。确实如此，然而在本教程中我们从未最终使用它，所以它只是作为数组保留在模式中。如果这是一个正式的架构设计，它可能最终会变成一个“多链接”，或者因为我们并不需要它而被删除掉。）

## 捕获 SELECT 表达式（Capturing a SELECT expression）

添加这些国家后，我们现在准备创建德古拉。首先我们先将 `Person` 中的 `places_visited` 由 `City` 改为 `Place` 使得它有更多可能性：伦敦（London），比斯特里察（Bistritz），匈牙利（Hungary）等等。 我们目前只是知道德古拉一直在罗马尼亚（Romania），所以当我们选择罗马尼亚时，我们可以做一个快速的 `FILTER`。这样做时，我们将 `SELECT` 放在 `()` 括号内。括号（圆括号）是捕获 `SELECT` 查询结果所必需的，然后我们可以使用结果来做一些事情。换句话说，括号界定了（设置边界）一个集合。EdgeDB 将在括号内进行操作，然后将完成的结果提供给 `places_visited`。

```edgeql
INSERT Vampire {
  name := 'Count Dracula',
  places_visited := (SELECT Place FILTER .name = 'Romania'),
  # .places_visited is the result of this SELECT query.
};
```

执行结果是：`{Object {id: 0a1b83dc-f2aa-11ea-9f40-038d228e2bba}}`

`uuid` 是来自服务器的回复，显示我们刚刚创建了哪个对象以及表明我们成功了。

让我们来检查一下 `places_visited` 是否有效。目前我们只是有一个 `Vampire` 的对象，所以让我们 `SELECT` 它：

```edgeql
SELECT Vampire {
  places_visited: {
    name
  }
};
```

执行结果是：`{Object {places_visited: {Object {name: 'Romania'}}}}`

完美！（王祖蓝.jpg）

## 添加约束（Adding constraints）

现在让我们再回过头关注一下 `age`。对于 `Vampire` 类型，这很容易，因为他们可以永生。但是现在我们也想给不可能永生的人类 `PC` 和 `NPC` 一个 `age`（我们不会期待他们能活 32767 年）。对于这一点，我们可以添加一个“约束（constrait）”（一个限制）。我们将给它们一个名为 `HumanAge` 的新类型，而不是 `age`。然后我们对其写下 `constraint` 并使用 [其中一个函数](https://edgedb.com/docs/datamodel/constraints) 。我们将使用
`max_value()`。 

下面是 `max_value()` 的签名：

`std::max_value(max: anytype)`

`anytype` 部分很有趣，因为这意味着它也可以处理像字符串这样的类型。例如，使用约束 `max_value('B')` 就不能使用“C”、“D”等。

现在让我们回到我们对 `HumanAge` 的约束，即 120。`HumanAge` 类型如下所示：

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
}
```

请记住，它是一个标量类型，因为它只能有一个值。然后我们将它添加到 `NPC` 类型。

```sdl
type NPC extending Person {
  property age -> HumanAge;
}
```

`HumanAge` 是我们自己的类型，有自己的名字，它是一个不能大于 120 的 `int16`。所以如果我们像下方一样输入，它将不起作用：

```edgeql
INSERT NPC {
    name := 'The innkeeper',
    age := 130
};
```

这里会出现一个错误提示：`ERROR: ConstraintViolationError: Maximum allowed value for HumanAge is 120.`

现在，如果我们将 `age` 更改为 30，我们会收到一条消息，表明它有效： `{Object {id: 72884afc-f2b1-11ea-9f40-97b378dbf5f8}}`。现在
没有 NPC（非玩家角色 Non-Player Character）可以超过 120 岁。

## 删除对象（Deleting objects）

在 EdgeDB 里删除是十分容易的：只需要使用关键字 `DELETE`。类似于 `SELECT`，当你写了 `DELETE` 然后是类型，默认情况下会删除该类型的所有。和 `SELECT` 一样，如果你使用 `FILTER` 那么它只会删除匹配过滤器的那些数据。

这种与 `SELECT` 的相似性可能会让您感到紧张，因为如果您键入类似 `SELECT City` 的内容，它将选择 `City` 的所有内容。`DELETE` 是相同的：`DELETE City` 会删除 `City` 类型的每一个对象。这就是为什么如果你没有使用 `FILTER` 做了删除，会弹出一个确认信息和你确认你真的想要删除这一切。但如果你使用了 `FILTER`，将不会有确认信息，因为被删除的内容不多且会假设你已经十分清楚你想要删除的内容。

所以让我们试一试。还记得我们创建的匈牙利（Hungary）和罗马尼亚（Romania）两个 `Country` 类型的对象吗？让我们删除它们：

```edgeql
DELETE Country;
```

就像插入一样，EdgeDB 返回为我们现在正被删除的对象的 id：

```
{Object {id: bc9c4766-2898-11eb-b5e8-0bfd25af166c}, Object {id: bd0a6b1a-2898-11eb-b5e8-ef85694d442f}}
```

让我们再把它们插入回去。现在让我们再来试一下带过滤的删除：

```edgeql
DELETE Country FILTER .name ILIKE '%States%';
```

没有任何匹配项，因此输出为 `{}` —— 我们没有删除任何内容。让我们再试一次：
Nothing matches, so the output is `{}` - we deleted nothing. Let's try again:

```edgeql
DELETE Country FILTER .name ILIKE '%ania%';
```

我们得到 `{Object {id: eaa9b03a-2898-11eb-b5e8-27121e63218a}}`，即 Romania。现在只剩下了 Hungary。如果我们想查看我们到底删除了什么怎么办？没问题 —— 只需将 `DELETE` 放在方括号内，然后 `SELECT` 即可。让我们再次删除所有 `Country` 对象，但这次我们将 `SELECT` 删除结果：

```edgeql
SELECT (DELETE Country) {
  name
};
```

输出结果是：`{Object {name: 'Hungary'}}`，这说明我们删除了 Hungary。如果现在我们执行 `SELECT Country`，我们将得到一个空集 `{}`，这说明我们确实把它们都删了。

（有趣的是：在 EdgeDB 里，`DELETE` 语句实际上是 `DELETE (SELECT ...)` 的 [语法糖](https://www.edgedb.com/docs/edgeql/statements/delete/)。另外，在下一章中，你将通过 `SELECT` 学习关键字 `LIMIT`，当你学习使用它时，请记住你也同样可以将其应用到 `DELETE`。）

> 语法糖（英语：Syntactic sugar）是由英国计算机科学家彼得·兰丁发明的一个术语，指计算机语言中添加的某种语法，这种语法对语言的功能没有影响，但是更方便程序员使用。语法糖让程序更加简洁，有更高的可读性。

最后，让我们再次插入匈牙利（Hungary）和罗马尼亚（Romania）以供后续使用。

[→ 点击这里查看第 3 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 此查询试图显示每个 `NPC` 的 `name`，并给每个 `NPC` 加上所有 `City` 类型，但执行后给出了错误。请问它缺少了什么？

   ```edgeql
   SELECT NPC {
     name,
     cities := SELECT City.name
   };
   ```

2. 如果 `City` 类型需要一个名为 `population` 的 `required property` 属性，它会是什么样子？“population”会是什么类型？
3. 此查询出于某种原因想要显示两次 `name`，但出现错误。你能想出办法吗？

   ```edgeql
   SELECT Person {
     name,
     name
   };
   ```

   （提示：问题是名称 `name` 被使用了两次）

4. 有人一直在尝试给某个角色赋予负数年龄，你可以想出一种阻止这种情况的约束吗？

   （提示：当前约束是`max_value(120);`）

5. 你能插入一个 HumanAge 类型吗？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森：“这个德古拉伯爵对历史太了解了！我很庆幸我来了。”_
