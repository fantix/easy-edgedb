---
tags: Type Annotations, Reverse Links
---

# 第十四章 - 一线希望

> 最终还是有一些好消息：乔纳森·哈克（Jonathan Harker）还活着。他逃离德古拉城堡（Castle Dracula）后，于 8 月找到了前往布达佩斯（Budapest）的路，然后进了一家医院，医院给米娜（Mina）发了一封信。医院告诉米娜：“他受到了一些可怕的刺激，不停地说着狼、毒和血，鬼和恶魔。” 米娜乘坐火车前往乔纳森所在的康复医院，他们乘火车回到英格兰（England），在埃克塞特（Exeter）举行了婚礼。米娜从埃克塞特寄给了露西一封信，告诉她这个好消息……但是太迟了，露西再也没有能打开它。与此同时，上一章提到的男人们按着计划到访了墓地，看到了吸血鬼露西在四处走动。当亚瑟（Arthur）看到她时，他终于相信了范海辛（Van Helsing），其他人也是如此。他们现在明白了吸血鬼是真实存在的，并设法摧毁了她。亚瑟很伤心，但很高兴看到露西不再被迫成为一只吸血鬼，现在可以平静地离开人世了。

所以我们有了一个名为“Exeter（埃克塞特）”的新城市，添加它很容易：

```edgeql
INSERT City {
  name := 'Exeter',
  population := 40000
};
```

40000 是当时埃克塞特的人口，且它没有一个与当时书中名称不同的 `modern_name`。

## 为类型添加注解及 @ 的使用（Adding annotations to types and using @）

既然我们知道如何进行内省查询，我们就可以开始给我们的类型 `annotations`（注解）了。注释是类型定义中的一个字符串，它为我们提供有关它的信息。默认情况下，注释可以使用标题 `title` 或 `description`。
Now that we know how to do introspection queries, we can start to give our types `annotations`. An annotation is a string inside the type definition that gives us information about it. By default, annotations can use the titles `title` or `description`.

假设在我们的游戏中，一个 `City` 至少需要 50 座建筑物。让我们对此使用 `description`：

```sdl
type City extending Place {
  annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
  property population -> int64;
}
```

现在我们可以对其进行 `INTROSPECT` 查询。我们从上一章就知道如何可以做到这一点 —— 只需在各处添加上 `: {name}` 即可获取其内部细节：

```edgeql
SELECT (INTROSPECT City) {
  name,
  properties: {name},
  annotations: {name}
};
```

哦，还不够完整：

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {schema::Annotation {name: 'std::description'}},
  },
}
```

当然：`annotations: {name}` 部分返回的是 _type_ 的名称，即 `std::description`。换句话说，它是一个链接，链接的目标只是告诉我们所使用的注释类型。但我们要找的是其中的“值”。

这就是 `@` 的用武之地。为了获得里面的值，我们要写入：`@value`。`@` 用于直接访问内部的值（字符串），而不仅仅是类型名称。让我们再试一次：

```edgeql
SELECT (INTROSPECT City) {
  name,
  properties: {name},
  annotations: {
    name,
    @value
  }
};
```

现在我们看到实际的注解了：

```
{
  schema::ObjectType {
    name: 'default::City',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'important_places'},
      schema::Property {name: 'modern_name'},
      schema::Property {name: 'name'},
      schema::Property {name: 'population'},
    },
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'Anything with 50 or more buildings is a city - anything else is an OtherPlace',
      },
    },
  },
}
```

如果我们想要一个除了 `title` 和 `description` 之外的具有不同名称的注释怎么办？这很容易，只需在架构中声明 `abstract annotation` 并为其命名即可。我们想添加一个警告，我们就这么叫它：

```sdl
abstract annotation warning;
```

假设 `Castle` 类型不仅仅用于城堡，也用于城堡镇，而不是对城堡镇用 `OtherPlace`。多亏了上面新加的抽象注解，现在 `OtherPlace` 可以用新的注解类型给出了更多的信息：

```sdl
type OtherPlace extending Place {
  annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
  annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
}
```

现在让我们仅对其名称和注释进行内省查询：

```edgeql
SELECT (INTROSPECT OtherPlace) {
  name,
  annotations: {name, @value}
};
```

这是结果：

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {name: 'std::description', @value: 'A place with under 50 buildings - hamlets, small villages, etc.'},
      schema::Annotation {name: 'default::warning', @value: 'Castles and castle towns do not count! Use the Castle type for that'},
    },
  },
}
```

## 更多关于日期（Even more working with dates）

现在有很多角色陆续死去了，所以让我们来考虑一下这个问题。我们可以运用 `cal::local_date` 来判断谁还活着，谁已经死了。首先让我们看看到目前为止我们拥有多少个 `People` 对象。我们可以使用 `SELECT count(Person)` 轻松算出，结果是 `{23}`。

还有一个名为 [`enumerate()`](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::enumerate) 的函数，它会给出由索引与输入集合组成的元组。我们将用它与我们的 `count()` 函数的结果进行比较，以确保我们得到的数字是正确的。

首先是一个如何使用 `enumerate()` 的简单示例：

```edgeql
WITH three_things := {'first', 'second', 'third'},
SELECT enumerate(three_things);
```

输出是：

```
{(0, 'first'), (1, 'second'), (2, 'third')}
```

所以现在让我们在 `SELECT enumerate(Person.name);` 里使用它，以确保我们有 23 个结果。最后一个索引应该是 22：

```
{
  (0, 'Renfield'),
  (1, 'The innkeeper'),
  (2, 'Mina Murray'),
# snip
  (14, 'Count Dracula'),
  (15, 'Woman 1'),
  (16, 'Woman 2'),
  (17, 'Woman 3'),
}
```

只有 18 个？哦，没错：因为 `Crewman` 对象没有名称，所以它们不会出现。那我们如何在查询中获取它们？我们当然可以像下面这样做：

```edgeql
WITH
  a := array_agg((SELECT enumerate(Person.name))),
  b := array_agg((SELECT enumerate(Crewman.number))),
SELECT (a, b);
```

（`array_agg()` 是为了避免发生集合乘以集合，正如我们在第 12 章中看到的那样）

但结果并不令人满意：

```
{
  (
    [
      (0, 'Renfield'),
      (1, 'The innkeeper'),
      (2, 'Mina Murray'),
# snip
      (14, 'Count Dracula'),
      (15, 'Woman 1'),
      (16, 'Woman 2'),
      (17, 'Woman 3'),
    ],
    [(0, 1), (1, 2), (2, 3), (3, 4), (4, 5)],
  ),
}
```

`Crewman` 类型现在只是数字，看起来不太好。让我们放弃像上面一样花哨的查询，用基于数字的名称来更新它们。这很简单：

```edgeql
UPDATE Crewman
SET {
  name := 'Crewman ' ++ <str>.number
};
```

顺便说一下，我们没有更多的 `Crewman` 类型要添加，但如果我们有，那么我们可以将添加时的结构改为下面这样，以避免之后需要 `UPDATE`:

```
type Crewman extending HasNumber, Person {
  overloaded property name := 'Crewman ' ++ <str>.number; #this part is new
}
```

现在每个人都有名字了，让我们用它来看看他们是否还在世。逻辑很简单：我们输入一个 `cal::local_date`，如果它大于 `last_appearance` 的日期，那么表明这个角色已经死了。

```edgeql
WITH p := (SELECT Person),
     date := <cal::local_date>'1887-08-16',
SELECT(p.name, p.last_appearance, 'Dead on ' ++ <str>date ++ '? ' ++ <str>(date > p.last_appearance));
```

这是输出：

```
{
  ('Lucy Westenra', <cal::local_date>'1887-09-20', 'Dead on 1888-08-16? true'),
  ('Crewman 1', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 2', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 3', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 4', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
  ('Crewman 5', <cal::local_date>'1887-07-16', 'Dead on 1888-08-16? true'),
}
```

如果我们用得足够多的话，我们当然可以把它变成一个函数。

## 反向链接（Reverse links）

最后，让我们看看如何反向跟踪链接，这是 EdgeDB 最强大和最有用的功能之一。学习使用它可能需要一些努力，但非常值得。

我们知道如何通过名称获取德古拉伯爵的 `slaves`，如下所示：

```edgeql
SELECT Vampire {
  name,
  slaves: {
    name
  }
};
```

输出结果如下所示：

```
{
  default::Vampire {
    name: 'Count Dracula',
    slaves: {
      default::MinorVampire {name: 'Woman 1'},
      default::MinorVampire {name: 'Woman 2'},
      default::MinorVampire {name: 'Woman 3'},
      default::MinorVampire {name: 'Lucy Westenra'},
    },
  },
}
```

但是如果我们想做相反的事情呢？即，从 `SELECT MinorVampire` 开始，并希望访问与其链接的 `Vampire` 对象。因为现在，我们只能调出属于 `MinorVampire` 和 `Person` 类型的属性。让我们考虑一下：

```edgeql
SELECT MinorVampire {
  name,
  # master... how do we get this?
  # There's no link to Vampire inside MinorVampire...
}
```

既然没有 `link master -> Vampire`，我们如何倒退查看链接到它的 `Vampire` 类型？

这就要靠反向链接了，我们使用 `.<` 代替 `.` 并指定我们要查找的类型：`[IS Vampire]`。

首先让我们抛开 `MinorVampire` 的查询，看看`.<` 是如何工作的。这有一个例子：

```edgeql
SELECT MinorVampire.<slaves[IS Vampire] {
  name,
  age
};
```

因为它以相反的顺序进行，所以它是指选择具有类型为 `MinorVampire` 的 `.slaves` 的 `Vampire`。

你可以将 `MinorVampire.<slaves[IS Vampire] {name, age}` 视为“选择‘拥有 MinorVampire 类型奴隶’的 Vampire 的名称和年龄” —— 从右到左。

这里是输出：

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```

到目前为止，这与 `SELECT Vampire: {name, age}` 相同。但它在前面的查询中非常有用，因为我们想访问多个类型。现在我们可以选择所有的 `MinorVampire` 类型的对象和它们的主人：

```edgeql
SELECT MinorVampire {
  name,
  master := .<slaves[IS Vampire] {name},
};
```

你可以将 `.<slaves[IS Vampire] {name}` 读作“通过 `.slaves` 链接到 `MinorVampire` 的 `Vampire` 的名称”。 

这里是输出：

```
{
  default::MinorVampire {
    name: 'Lucy Westenra',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 1',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 2',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
  default::MinorVampire {
    name: 'Woman 3',
    master: {default::Vampire {name: 'Count Dracula'}},
  },
}
```

[→ 点击这里查看第 14 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何仅显示所有 `Person` 对象的编号？比如，如果有 20 个，则显示 `1, 2, 3..., 18, 19, 20`。

2. 使用反向查找，你将如何显示 1）所有名称中带有 `o` 的 `Place` 的对象（及他们的名字）；2）访问过这些地方的人的名字？

3. 使用反向查找，你将如何显示所有后来成为了 `MinorVampire` 的 `Person` 对象？

   提示：别忘了，`MinorVampire` 有一个指向自己成为吸血鬼之前的前身的链接。

4. 如何给 `MinorVampire` 类型一个名为 `note` 的注解，说明 `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`？

5. 如何在查询中看到 `MinorVampire` 的 `note` 注释？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候报仇了。_
