---
tags: Filtering On Insert, Json
---

# 第六章 - 还是逃不掉

> 三个女吸血鬼就在乔纳森的身边，但他无法动弹。突然，德古拉跑进了房间，让这些女人离开：“之后你们可以拥有他，但今晚不行！”。女吸血鬼们听从并离开了。乔纳森时醒来躺在他的床上，感觉就像做了一场噩梦……但看有人叠好了他的衣服，他就知道这不会只是一个梦。明天会有一些来自斯洛伐克的游客参观城堡，所以乔纳森想了个主意。他写了两封信，一封给米娜，一封给他的老板。他给了来访者一些钱，请求他们为他寄信。但被德古拉发现了，他很生气。他在乔纳森面前烧毁了信件，并警告他不许再犯。乔纳森仍然被困在城堡里，德古拉知道他仍然在想办法试图逃跑。

## 插入时过滤集合（Filtering on sets when doing an insert）

本章中没有太多关于类型的新内容，所以让我们来看看如何改进我们的架构（schema）。现在对乔纳森·哈克（Jonathan Harker）的插入语句仍然是这样的：

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

当我们只有城市时，这是没问题的。但现在我们有了 `Place` 和 `Country` 两个类型。首先，我们先来插入两个新的 `Country` 对象以增加多样性：

```edgeql
INSERT Country {
  name := 'France'
};
INSERT Country {
  name := 'Slovakia'
};
```

（在第 9 章中，我们将学习如何只用一个 `INSERT` 来做到这同时插入多个对象！）

现在，我们为既不是城市也不是国家的地方创建一个名为 `OtherPlace` 的新类型。这很容易做到：`type OtherPlace extending Place;`。

然后，插入我们的第一个 `OtherPlace`：

```edgeql
INSERT OtherPlace {
  name := 'Castle Dracula'
};
```

`OtherPlace` 为我们提供了大量来自 `Place` 类型且不属于 `City` 类型的地方。

回到乔纳森：在我们的数据库中，他去过四个城市、一个国家和一个 `OtherPlace`……但他没有去过斯洛伐克（Slovakia）或法国（France），所以我们不能直接用 `places_visited := SELECT Place` 对其进行插入。但我们可以根据他访问过的地方的名称对 `Place` 进行过滤。像这样：

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name IN {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};
```

你会留意到我们只是使用 `{}` 在集合中写入了这些地方的名称，我们不需要使用带有 `[]` 的数组来执行此操作。（顺便说一下，这称为 [set 构造函数](https://www.edgedb.com/docs/edgeql/expressions/overview/#set-constructor)。）

现在如果乔纳森逃离了德古拉城堡并来到了一个新地方怎么办？让我们假装他逃跑了并逃到了斯洛伐克（Slovakia）。当然，我们可以修改他的 `INSERT`，在造访点的名字集合里加上 `'Slovakia'`。但是我们如何才能做一次快速的更新呢？对此，我们有关键字 `UPDATE` 和 `SET`。`UPDATE` 要更新的类型，`SET` 我们要修改的部分。像这样：

```edgeql
UPDATE NPC
FILTER .name = 'Jonathan Harker'
SET {
  places_visited += (SELECT Place FILTER .name = 'Slovakia')
};
```

如果语句执行成功了，EdgeDB 将返回更新成功的对象的 ID。在上面的例子里，仅仅会返回一个 ID（因为只有一个对象被更新）：

```
{
  Object { id: <uuid>"6f436006-3f65-11eb-b6de-a3e7cc8efd4f" }
}
```

如果我们写了类似 `FILTER .name = 'SLLLlovakia'` 的过滤，那么 EdgeDB 会返回 `{}`，让我们知道没有匹配到任何。或者更准确地说：顶级对象在 `FILTER .name = 'Jonathan Harker'` 上得到匹配，但 `places_visited` 并没有得到更新，因为 `FILTER` 里没有获得任何可匹配的对象。

由于乔纳森（Jonathan）还没有访问过斯洛伐克（Slovakia），我们现在可以使用 `-=` 代替 `+=`，并使用相同的 `UPDATE` 语法来删除它。

现在我们了解了在 `SET` 之后可以使用的 [所有三个运算符](https://www.edgedb.com/docs/edgeql/statements/update) ：`:=`、`+=` 和`-=`。 

让我们来做另一个更新。还记得这个吗？

```edgeql
SELECT Person {
  name,
  lover
} FILTER .name = 'Jonathan Harker';
```

米娜·默里（Mina Murray）的 `lover` 是乔纳森·哈克（Jonathan Harker），但因为我们先插入了乔纳森，所以乔纳森没有 `lover`。我们现在可以更新它：

```edgeql
UPDATE Person FILTER .name = 'Jonathan Harker'
SET {
  lover := (SELECT Person FILTER .name = 'Mina Murray' LIMIT 1)
};
```

现在，Jonathan 的 `link lover` 终于显示了 Mina 而不再是空的 `{}`。

当然，如果你在没有 `FILTER` 的情况下使用了 `UPDATE`，它将对所有对象进行相同的更改。例如，下面的语句会在数据库里把所有 `Place` 更新进每一个 `Person` 对象的 `places_visited` 属性中：

```edgeql
UPDATE Person
SET {
  places_visited := Place
};
```

## 级联运算符 ++（Concatenation with ++）

还有一个运算符是 `++`，它执行级联（连接在一起）而不是做“相加”。

你可以像这样：`SELECT 'My name is ' ++ 'Jonathan Harker';` 做简单的操作，从而得到结果 `{'My name is Jonathan Harker'}`。或者你可以做更复杂的级联，只要你继续将字符串连接到字符串：

```edgeql
SELECT 'A character from the book: ' ++ (SELECT NPC.name) ++ ', who is not ' ++ (SELECT Vampire.name);
```

输出为：

```
{
  'A character from the book: Jonathan Harker, who is not Count Dracula',
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
}
```

（这个级联运算符也适用于数组，它可以将被操作的数组合并放入到一个数组中。所以执行 `SELECT ['I', 'am'] ++ ['Jonathan', 'Harker'];` 的结果是 `{['I', 'am', 'Jonathan', 'Harker']}`。）

现在，让我们再来更改一下 `Vampire` 类型，使其也拥有链接可以指向被其掌控的 `MinorVampire`。你应该记得德古拉伯爵是唯一一个真正的吸血鬼，而其他吸血鬼都是 `MinorVampire` 类型。这意味着我们需要一个 `multi link`：

```sdl
type Vampire extending Person {
  property age -> int16;
  multi link slaves -> MinorVampire;
}
```

然后我们可以在插入德古拉伯爵（Count Dracula）信息的同时 `INSERT` 相关的 `MinorVampire` 对象。但首先让我们先从 `MinorVampire` 中删除 `required link master`，因为我们不希望两个对象相互链接。原因有二：

- 会使事情变得复杂。假设我们声明的 `Vampire` 里包含一个指向 `MinorVampire` 的 `slaves` 链接，在我们插入 `Vampire` 时，如果我们还没有创建对应的 `MinorVampire`，那么它将是空 `{}`，则我们将不得不在创建 `MinorVampire` 后，再对 `Vampire` 进行更新；如果我们先创建的是 `MinorVampire`，且它有一个指向 `Vampire` 的 `master` 链接，而我们还没有创建 `Vampire`，那么这个 `master` 将不存在，尤其当它是个 `required link` 时，我们必须提供一个 `master`。
- 如果这两种类型相互链接，我们将无法在需要时删除它们。错误信息如下所示：

```
edgedb> delete MinorVampire;
ERROR: ConstraintViolationError: deletion of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114) is prohibited by link target policy
  Detail: Object is still referenced in link slave of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6).
edgedb> delete Vampire;
ERROR: ConstraintViolationError: deletion of default::Vampire (e5ef5bc6-006f-11ec-93a9-77e907d251d6) is prohibited by link target policy
  Detail: Object is still referenced in link master of default::MinorVampire (ee6ca100-006f-11ec-93a9-4b5d85e60114).
```

因此，我们先简单地将 `MinorVampire` 改为没有 `link master` 的 `Person` 扩展类型：

```sdl
type MinorVampire extending Person {
}
```

然后我们在创建德古拉伯爵时，也一同创建她们，如下所示：

```edgeql
INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (INSERT MinorVampire {
      name := 'Woman 1',
    }),
    (INSERT MinorVampire {
      name := 'Woman 2',
    }),
    (INSERT MinorVampire {
      name := 'Woman 3',
    }),
  }
};
```

请注意两件事：（1）我们在 `slaves` 后面使用了 `{}`，并把所有要插入的 `INSERT` 都放到了这个集合；（2）每一个 `INSERT` 都放在了小括号 `()` 里去捕捉插入。

现在我们不必像以前一样，先插入 `MinorVampire` 类型，然后再通过过滤并更新给 `Vampire`：我们可以将她们与德古拉（Dracula）放在一起插入。

然后当我们 `SELECT Vampire` 时：

```edgeql
SELECT Vampire {
  name,
  slaves: {name}
};
```

我们可以由此得到想要的输出结果，将德古拉（Dracula）和女吸血鬼们一并显示出来：

```
Object {
  name: 'Count Dracula',
  slaves: {Object {name: 'Woman 1'}, Object {name: 'Woman 2'}, Object {name: 'Woman 3'}},
},
```

这可能会使你好奇：如果我们就是想建立双向链接怎么办？实际上这里确实有一种非常方便的方法（称为 **向后链接**），但我们要到第 14 章和第 15 章才会看它。如果你真的很好奇，你可以直接跳到后面的章节，但在那之前我们还有很多东西要学。

## 用 \<json> 生成 JSON（Just type \<json> to generate json）

如果我们想要输出 JSON 格式，该怎么做？再简单不过了：使用 `<json>` 进行转换即可。EdgeDB 中的任何类型（`bytes` 除外）都可以轻松转换为 JSON：

```edgeql
SELECT <json>Vampire {
  # <json> is the only difference from the SELECT above
  name,
  slaves: {name}
};
```

输出结果为:

```
{
  "{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Woman 1\"}, {\"name\": \"Woman 2\"}, {\"name\": \"Woman 3\"}]}",
}
```

## 从 JSON 转换回来（Converting back from JSON）

那么反过来呢，就是把 JSON 转回成 EdgeDB 的类型呢？这也是可行的，但要当心 JSON 数据的类型，因为 EdgeDB 讲究转换的对称性：转成 JSON 前是什么类型，转回来还应该是什么类型。比如，《德古拉》里出现的第一个日期字符串 `'18870503'`，如果我们尝试将它的 JSON 值转换为一个 `int64`，它将无法正常工作：

```edgeql
SELECT <int64><json>'18870503';
```

问题出在从 JSON 字符串到 EdgeDB `int64` 的转换，错误提示为：`ERROR: InvalidValueError: expected json number, null; got json string`。也就是说，EdgeDB `str` 转换为 JSON 字符串后，又试图转换为 EdgeDB `int64`，这是不可行的。（除非是 EdgeDB 数字类型转换为 JSON 数字，才可再转换回 EdgeDB 数字类型。）因此，这里为了保持对称，你需要先将 JSON 字符串转换为 EdgeDB 的 `str`，然后再转换为 `int64`：

```edgeql
SELECT <int64><str><json>'18870503';
```

现在它正常工作了：我们可以得到 `{18870503}`，过程是 EdgeDB `str`，变成了 JSON 字符串，然后又回到 EdgeDB `str`，最后被转换为 `int64`。

接下来，我们再来看下面的列子，即把《德古拉》里出现的第一个日期字符串转换成 JSON 字符串后再转换回成 `cal::local_date` 类型：

```edgeql
SELECT <cal::local_date><json>'18870503';
```

你会发现它是可以正确执行的，结果是 `{<cal::local_date>'1887-05-03'}`。因为 `<json>` 将 `'18870503'` 转换为了 JSON 字符串，且 `cal::local_date` 可以接收 JSON 字符串以进行创建。换句话说，当你对 EdgeDB 的 `cal::local_date` 进行 JSON 转换时，你将得到一个 JSON 字符串，因此反之，你固然可以将符合日期格式的 JSON 字符串直接转换回 `cal::local_date`。

[关于 JSON 的文档](https://www.edgedb.com/docs/datamodel/scalars/json) 解释了哪些 JSON 类型可以转换为哪些 EdgeDB 类型，如果你需要对 JSON 进行大量转换，可以将其添加至书签，便于后续查阅。你还可以在 [此处](https://www.edgedb.com/docs/edgeql/funcops/json) 查看到所有 JSON 的函数列表。

[→ 点击这里查看到第 6 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 下面这个选择是不完整的。如何修改它从而使它能打印出“Pleased to meet you, I'm ”以及 NPC 的名字？

   ```edgeql
   SELECT NPC {
     name,
     greeting := ## Put the rest here
   };
   ```

2. 如果米娜要去德古拉城堡参观，你会如何更新米娜（Mina）的 `places_visited`，让它也包括罗马尼亚？

3. 如何显示出名称（name）里包含 `{'W', 'J', 'C'}` 中任何一个大写字母的所有 `Person` 对象？

   提示：使用 `WITH` 和级联 `++`（concatenation）操作。 

4. 如何用 JSON 显示和上一题相同的查询？

5. 如何将“ the Greate”添加到每个 Person 类型的对象中？

   此外，使用字符串索引来撤消此操作的快速方法是什么？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森爬上城墙进入德古拉的房间。_
