---
tags: Introspection, Type Union Operator
---

# 第十三章 - 认识新的露西

> 这次救露西（Lucy）已经来不及了，她快要死了。突然她睁开了眼睛 —— 看起来很奇怪。她看着亚瑟（Arthur）说：“亚瑟！亲爱的，我很高兴你来了！请吻我！” 他正要去吻她，但范海辛（Van Helsing）抓住他说：“你敢！” 说话的不是露西，而是她身体里面的吸血鬼。她已经死了，范海辛把一个金色的十字架放在她的嘴唇上以阻止她移动（十字架对吸血鬼有这种力量）。不幸的是，护士趁没人看见时候偷了十字架去卖。几天后，有消息称一个女性正在偷窃和咬孩子 —— 是吸血鬼露西。报纸称它为“Bloofer Lady”，因为年幼的孩子们试图称她为“Beautiful Lady”（美丽的女士），但不能正确发音 _beautiful_。现在范海辛把吸血鬼的真相告诉了其他人，但亚瑟不相信他，并且非常生气他用如此疯狂的事情诋毁自己的妻子。范海辛说：“好吧，你不相信我。那让我们今晚一起去墓地看看会发生什么。也许到时候你就相信了。” 

看起来原本是 `NPC` 的露西已经变成 `MinorVampire`（小吸血鬼）了。我们该如何在数据库里展示这一点呢？让我们先再过一遍这些类型。

现在的 `MinorVampire` 并没什么特别的，只是一个扩展自 `Person` 的类型：

```sdl
type MinorVampire extending Person {
}
```

现在，根据书中的描述，露西给人类带来了一个新“类型”。旧的露西已经消失了，新的露西现在是一个名为德古拉伯爵的 `Vampire`（吸血鬼）的 `slaves`（奴隶）。

因此，与其尝试更改 `NPC` 类型，我们只需给 `MinorVampire` 一个指向 `Person` 的可选链接：

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
}
```

之所以设置为可选的，是因为我们并不一定知道他们成为吸血鬼之前是谁。例如，我对德古拉控制的那三个女吸血鬼之前的身世一无所知，因此我们无法为她们制作 `NPC` 类型来尝试链接到属性 `former_self`。

另一种（非正式地）链接它们的方法是给 `NPC` 的 `last_appearance` 和 `MinorVampire` 的 `first_appearance` 设置相同的日期。首先，我们先更新 `NPC` 露西的 `last_appearance`：

```edgeql
UPDATE Person filter .name = 'Lucy Westenra'
SET {
  last_appearance := cal::to_local_date(1887, 9, 20)
};
```

然后我们可以将露西（Lucy）添加到德古拉（Dracula）的 `INSERT` 中。（如果前面章节里的操作你都有执行，那么现在只需先 `DELETE Vampire;` 和 `DELETE Minorvampire;`，然后我们就可以再一次练习 `INSERT` 了。）

注意第一行，我们创建了一个名为 `lucy` 的变量。然后我们使用它来引入所有数据，创建她成为 `MinorVampire`，这比手动插入所有信息要高效得多。这里还需要包括露西的力量：我们给它额外加 5，因为吸血鬼更加强壮些。

这里是具体插入：

```edgeql
WITH lucy := (SELECT Person filter .name = 'Lucy Westenra' LIMIT 1)
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
    (INSERT MinorVampire {
      name := lucy.name,
      former_self := lucy,
      first_appearance := lucy.last_appearance,
      strength := lucy.strength + 5,
    }),
  },
  places_visited := (SELECT Place FILTER .name in {'Romania', 'Castle Dracula'})
};
```

多亏了 `former_self` 链接，我们很容易找到所有来自 `Person` 对象的小吸血鬼。只需用 `EXISTS previous_self` 进行过滤：

```edgeql
SELECT MinorVampire {
  name,
  strength,
  first_appearance,
} FILTER EXISTS .former_self;
```

输出结果：

```
{
  Object {
    name: 'Lucy Westenra',
    strength: 7,
    first_appearance: <cal::local_date>'1887-09-20',
  },
}
```

使用其他的过滤方式也可以，比如 `FILTER .name IN Person.name AND .first_appearance IN Person.last_appearance;` 也是可以的，但检查链接是否 `EXISTS`（存在）最简单。我们也可以把涉及时间的类型转换为 `cal::local_datetime`，而不是 `cal::local_date`，以获得精确到分钟的准确时间。但是我们现在还不需要那么精确。

## 类型联合运算符（The type union operator: |）

另一个与类型相关的运算符是 `|`，用于组合类型（类似于 `OR`）。例如，此查询拉出了所有 `Person` 类型用以判断露西所属类型是否在其中，并返回“真”：

```
SELECT (SELECT Person FILTER .name = 'Lucy Westenra') IS NPC | MinorVampire | Vampire;
```

意思是如果选择的 `Person` 类型是 `NPC` 或 `MinorVampire` 或 `Vampire` 其中的一个类型，则返回“真”。由于露西的 `NPC` 身份和露西的 `MinorVampire` 身份可以分别匹配到这三种类型中的一种，所以返回值是 `{true, true}`。

你还可以在你的架构（schema）里将类型联合运算符添加到链接中，这非常酷。例如，让我们假设游戏中还有其他 `Vampire` 对象，其中一个 `Vampire` 非常强大，可以控制另一个。但我们当前的架构里 `Vampire` 只能控制 `MinorVampire`，不能控制 `Vampire`：

```
type Vampire extending Person {
  multi link slaves -> MinorVampire;
}
```

因此，为了展现此更改，你可以使用 `|` 并添加另一种类型：

```
type Vampire extending Person {
  multi link slaves -> MinorVampire | Vampire;
}
```

目前，我们的数据库中只有德古拉伯爵（Count Dracula）作为 `Vampire`，因此我们并不打算真的以上面这种方式更改我们的架构，但请记住这个 `|` 运算符，也许有天你会需要它。

## 当目标被删除（On target delete）

我们在这里决定为露西保留旧的 `NPC` 对象，因为露西将在游戏中待到 1887 年 9 月，也许之后会有 `PC` 类型的对象与她互动。但这可能会让你想到关于链接的删除。即如果我们在她成为 `MinorVampire` 时想删除旧类型对象，会怎样？或者更现实地说，如果吸血鬼死了，所有连接到 `Vampire` 的 `MinorVampire` 都应该被删除，该怎么办？我们不会在我们的游戏中真的这样做，但你确实可以使用 `on target delete` 来实现。`on target delete` 的意思是“当目标被删除时”，执行链接声明后 `{}`里的内容。为此，我们有 [四个选项](https://www.edgedb.com/docs/datamodel/links#deletion)：

- `restrict`：禁止你删除目标对象。

所以如果你像这样声明 `MinorVampire`：

```sdl
type MinorVampire extending Person {
  link former_self -> Person {
    on target delete restrict;
  }
}
```

那么一旦 `NPC` 露西连接到 `MinorVampire` 露西，你就无法删除她（`NPC` 露西）。

- `delete source`：在这种情况下，删除 `Person` 露西（链接的目标）将自动删除 `MinorVampire` 露西（链接的来源）。

- `allow`：这个只是简单地允许你删除目标（这是默认设置）。

- `deferred restrict`：禁止你删除目标对象，除非它在事务（transaction）结束时不再是目标对象。因此，这个选项类似于 `restrict`，但具有更大的灵活性。

所以如果你想让所有的 `MinorVampire` 对象在它们的 `Vampire` 死亡时自动被删除，你可以给 `MinorVampire` 添加一个指向 `Vampire` 类型的链接。然后你可以使用 `on target delete delete source`：`Vampire` 是链接的目标，`MinorVampire` 是被删除的源。

现在让我们看一些查询的技巧。

## 使用 DISTINCT（Using DISTINCT）

`DISTINCT` 很简单：只需将 `SELECT` 更改为 `SELECT DISTINCT` 即可获得去重的结果。我们现在可以看到，如果我们执行 `SELECT Person.strength;`，那么对于我们的 `Person` 对象们，会得到相当多的重复项。它看起来会是这样：

```
{5, 4, 4, 4, 4, 4, 10, 2, 2, 2, 2, 2, 2, 2, 7, 5}
```

将其更改为 `SELECT DISTINCT Person.strength;`，输出将为 `{2, 4, 5, 7, 10}`。

`DISTINCT` 按项目（item）工作，不做解包，因此 `SELECT DISTINCT {[7, 8], [7, 8], [9]};` 返回的是 `{[7, 8], [9]}` 而不是 `{7, 8, 9}`。

## 使用 `__type__`（Getting `__type__` all the time）

我们之前已经看到，我们可以使用 `__type__` 来获取查询中的对象类型，并且 `__type__` 总是有 `.name` 来显示类型的名称（否则我们只会得到 `uuid`）。就像我们可以通过 `SELECT Person.name` 获得所有名字一样，我们可以像这样获得所有类型名：

```edgeql
SELECT Person.__type__ {
  name
};
```

它向我们展示了至今为止扩展自 `Person` 的所有类型：

```
{
  Object {name: 'default::Person'},
  Object {name: 'default::MinorVampire'},
  Object {name: 'default::Vampire'},
  Object {name: 'default::NPC'},
  Object {name: 'default::PC'},
  Object {name: 'default::Crewman'},
  Object {name: 'default::Sailor'},
}
```

或者我们也可以在常规查询中使用它来返回类型。让我们看看有哪些类型下有名为 `Lucy Westenra` 的对象：

```edgeql
SELECT Person {
  __type__: {
    name
  },
  name
} FILTER .name = 'Lucy Westenra';
```

这向我们展示了匹配的对象，当然它们是 `NPC` 和 `MinorVampire`。

```
{
  Object {__type__: Object {name: 'default::NPC'}, name: 'Lucy Westenra'},
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Lucy Westenra'},
}
```

这里有一个设置可以用来在查询时始终查看类型：只需键入 `\set introspect-types on`。一旦你这样做了，你将始终看到类型名称，而不仅仅是 `Object`。现在，即使是像这样的简单搜索也会为我们提供类型：

```edgeql
SELECT Person {
  name
} FILTER .name = 'Lucy Westenra';
```

这是输出：

```
{default::NPC {name: 'Lucy Westenra'}, default::MinorVampire {name: 'Lucy Westenra'}}
```

因为它非常方便，所以从现在开始，本书将展示使用了 `\set introspect-types on` 给出的结果。

## 内省（Being introspective）

我们刚刚在 `\set introspect-types on` 中使用的 `introspect` 这个词，也是它自己的关键字：`INTROSPECT`。每种类型都有我们可以访问的以下字段：`name`、`properties`、`links` 和 `target`，并且 `INTROSPECT` 可以让我们看到它们。让我们试一下，看看会得到了什么。我们将从我们的 `Ship` 类型开始，它很小，但包含前面提到的四个字段。为了避免我们已经忘了，这里再次列出 `Ship` 的属性和链接：

```sdl
type Ship {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

首先，这是最简单的 `INTROSPECT` 查询：

```edgeql
SELECT (INTROSPECT Ship);
```

这个查询对我们来说不是很有用，但它确实展示了它是如何工作的：它返回了 `{'default::Ship'}`。请注意，`INTROSPECT` 和类型放在括号内；它是之后你要捕捉的类型的 `SELECT` 表达式。

现在让我们将 `name`、`properties` 和 `links` 放入自省（introspection）：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links,
};
```

结果是：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 4332bd76-0134-11eb-b20a-777e9cc80030},
      schema::Property {id: 43379841-0134-11eb-a427-dd28c7eae0ce},
    },
    links: {
      schema::Link {id: 4339bd5f-0134-11eb-bdca-21c31be0f932},
      schema::Link {id: 43360bf2-0134-11eb-b89c-a1211cb5d40e},
      schema::Link {id: 43383b6f-0134-11eb-86db-17f19f35ac2f},
    },
  },
}
```

就像在类型上使用 `SELECT` 一样，如果输出包含另一种类型、属性等，我们只会得到一个 id。我们还是要指定我们想要什么。

因此，让我们在查询中添加更多内容以获取我们想要的信息：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {
    name,
    target: {
      name
    }
  },
  links: {
    name,
    target: {
      name
    },
  },
};
```

所以这将给出：

1. `Ship` 的类型名称，
2. 属性及其名称。But we also use `target`, which is what a property points to (the part after the `->`). For example, the target of `property name -> str` is `std::str`. And we want the target name too; without it we'll get an output like `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}`.
3. The links and their names, and the targets to the links...and the names of _their_ targets too.

With all that together, we get something readable and useful. The output looks like this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: 'crew', target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: '__type__', target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'sailors', target: schema::ObjectType {name: 'default::Sailor'}},
    },
  },
}
```

This type of query seems complex but it is just built on top of adding things like {name} every time you get output that only a machine can understand.

Plus, if the query isn't too complex (like ours), you might find it easier to read without so many new lines and indentation. Here's the same query written that way, which looks much simpler now:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {name, target: {name}},
  links: {name, target: {name}},
};
```

[→ 点击这里查看第 13 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. How would you insert an `NPC` named 'Mr. Swales' who has visited the `City` called 'York', the `Country` called 'England', and the `OtherPlace` called 'Whitby Abbey'? Try it in a single insert.

2. How readable is this introspect query?

   ```edgeql
   SELECT (INTROSPECT Ship) {
     name,
     properties,
     links
   };
   ```

3. What would be the shortest way to see what links from the `Vampire` type?

4. What do you think the output of `SELECT DISTINCT {1, 2} + {1, 2};` will be?

   Hint: don't forget the Cartesian multiplication.

5. What do you think the output of `SELECT DISTINCT {2, 2} + {2, 2};` will be?

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _一位老朋友回来了。_
