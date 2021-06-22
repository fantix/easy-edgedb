---
tags: Multiple Inheritance, Polymorphism
---

# 第八章 - 德古拉跨乘船前往英格兰

本章我们终于离开了德古拉城堡。这是本章中发生的事情：

> 一艘船从保加利亚（Bulgaria）的瓦尔纳市（Varna）出发，驶入黑海。它有一个**船长（captain）、大副（first mate）、二副（second mate）、厨师（cook）** 和 **五名船员（five crew）**。德古拉也在床上，但没人知道他在。每天晚上德古拉都会离开他的棺材，且每天晚上都会有一个人消失。船上的人感到害怕，但不知道发生了什么或该做什么。其中一个人说他看到一个奇怪的人在甲板上走来走去，但其他人不相信他。终于是船抵达英格兰（England）惠特比市（Whitby）的最后一天了，只剩下船长一个人了 —— 其他所有人都消失了。船长现在知道真相了。他将双手绑在方向盘上，这样即使德古拉找到了他，船也能继续直行。第二天，惠特比（Whitby）的人们看到一艘船搁浅了，一只狼从上面跳下来并跑到了岸上 —— 这是德古拉的狼化身，但人们并不知道。人们发现死去的船长被绑在方向盘上，手里拿着一个笔记本，于是开始阅读里面的故事。

> 与此同时，米娜（Mina）和她的朋友露西（Lucy）正在惠特比（Whitby）度假……

## 多重继承（Multiple inheritance）

在德古拉到达惠特比（Whitby）时，让我们来学习多重继承（multiple inheritance）。我们知道你可以在一个类型上 `extend` 另一个类型，我们已经在之前如此做了很多次：`Person` 扩展出 `NPC`, `Place` 扩展出 `City`，等等。多重继承是同时对多个类型执行此操作。让我在船员身上做一下尝试。原著里没有给他们任何名字，所以我们用编号来表示他们。大多数 `Person` 类型不需要数字，因此我们将为需要数字的类型创建一个名为 `HasNumber` 的抽象类型：

```sdl
abstract type HasNumber {
  required property number -> int16;
}
```

我们还要从 `Person` 类型的 `name` 中删除 `required`。因为现在不是每个 `Person` 类型的对象都会有一个名字，对于有名字的 `Person`，我们信任自己会输入对应的名字。当然，我们还会保留 `exclusive`。

现在我们可以运用多重继承来创建 `Crewman` 类型。这很简单：只需在要扩展的每个类型之间添加一个逗号。

```sdl
type Crewman extending HasNumber, Person {
}
```

现在我们有了 `Crewman` 并且不需要名字，且 `count()`，是我们对船员的插入变得非常容易。我们只需要重复做五次：

```edgeql
INSERT Crewman {
  number := count(DETACHED Crewman) + 1
};
```

因此，如果还没有 `Crewman` 类型的对象，新插入的船员将获得编号 1。下一个将获得编号 2，依此类推。所以这样做五次之后，我们可以 `SELECT Crewman {number};` 来查看结果了。我们将得到：

```
{
  Object {number: 1},
  Object {number: 2},
  Object {number: 3},
  Object {number: 4},
  Object {number: 5},
}
```

接下来是 `Sailor` 类型。水手是有等级的，所以首先我们先为此创建一个枚举：

```sdl
scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;
```

然后我们将创建一个使用了 `Person` 和这个 `Rank` 枚举的 `Sailor` 类型：

```sdl
type Sailor extending Person {
  property rank -> Rank;
}
```

然后我们来制作一个 `Ship` 类型来承载它们：

```sdl
type Ship {
  required property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

现在要插入水手，我们只需给他们一个名字并从枚举中选择一个等级：

```edgeql
INSERT Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

INSERT Sailor {
  name := 'Petrofsky',
  rank := 'FirstMate'
};

INSERT Sailor {
  name := 'The Second Mate',
  rank := 'SecondMate'
};

INSERT Sailor {
  name := 'The Cook',
  rank := 'Cook'
};
```

插入 `Ship` 很容易，因为现在的每个 `Sailor` 和每个 `Crewman` 都是这艘船的一部分 —— 我们不需要使用任何 `FILTER`。

```edgeql
INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

然后我们可以查看 `Ship` 以确保全部船员都在里面：

```edgeql
SELECT Ship {
  name,
  sailors: {
    name,
    rank,
  },
  crew: {
    number
  },
};
```

The result is:

```
{
  Object {
    name: 'The Demeter',
    sailors: {
      Object {name: 'Petrofsky', rank: FirstMate},
      Object {name: 'The Second Mate', rank: SecondMate},
      Object {name: 'The Cook', rank: Cook},
      Object {name: 'The Captain', rank: Captain},
    },
    crew: {
      Object {number: 1},
      Object {number: 2},
      Object {number: 3},
      Object {number: 4},
      Object {number: 5},
    },
  },
}
```

## 序列类型（The sequence type）

关于给对象编号的情况，EdgeDB 有一个叫做 [sequence](https://www.edgedb.com/docs/datamodel/scalars/numeric/#type::std::sequence) 的类型，你可能会发现它很有用。这种类型被定义为“int64 的自动递增序列”，因此一个 `int64` 从 1 开始，每次使用时都会增加。让我们假设一个 `Townsperson` 类型，并对其使用 sequence。但下面这样做是错误的：

```sdl
type Townsperson extending Person {
  property number -> sequence;
}
```

这是行不通的，因为每个 `sequence` 都会记录最新的数字，如果每种类型都使用 `sequence`，那么它们就会共享它。所以正确的做法是将其扩展为你命名的另一种类型，然后该类型将从 1 开始计数。所以我们的 `Townsperson` 类型看起来应该像这样：

```sdl
scalar type TownspersonNumber extending sequence;

type Townsperson extending Person {
  property number -> TownspersonNumber;
}
```

即使你删除已经插入的项目，`sequence` 类型的数字也会继续增加 1。比如，如果您插入五个 `Townsperson` 对象，它们的数字将是从 1 到 5。然后，如果您将它们全部删除，然后有插入了一个 `Townsperson`，那么这个数字将会是 6（而不是 1）。所以对于我们的 `Crewman` 类型的创建，可能有另一种可能的选择。它非常方便，并且没有重复的可能，但是每次插入时数字都会自行增加 1。好吧，你 _可以_ 使用 `UPDATE` 和 `SET` 创建重复的数字（EdgeDB 不会阻止你），但即便如此，当你进行下一次插入时它仍然会跟踪下一个数字。

## 使用 IS 查询多类型（Using IS to query multiple types）

所以现在我们有相当多的类型扩展自 `Person` 类型，其中很多都有自己的属性。`Crewman` 类型有 `number` 属性，同时 `NPC` 类型有一个叫做 `age` 的属性。

如果我们想同时查询它们，该怎么办呢？他们都扩展自 `Person`，但 `Person` 没有它们的所有链接和属性。所以下面这个查询并不起作用：

```edgeql
SELECT Person {
  name,
  age,
  number,
};
```

错误提示是：`ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`。

幸运的是，有一个简单的解决方法：我们可以在方括号内使用 `IS` 来指定类型。具体做法是：

- `.name`：这个可以保持不变，因为 `Person` 有这个属性
- `.age`：这属于 `NPC` 类型，所以将其更改为 `[IS NPC].age`
- `.number`：这属于 `Crewman` 类型，因此将其更改为 `[IS Crewman].number`

现在它可以工作了：

```edgeql
SELECT Person {
  name,
  [IS NPC].age,
  [IS Crewman].number,
};
```

输出量相当大，所以这里只展示其中的一部分。你会注意到没有属性或链接的类型将返回一个空集：`{}`。

```
{
  Object {name: 'Woman 1', age: {}, number: {}},
  Object {name: 'The innkeeper', age: 30, number: {}},
  Object {name: 'Mina Murray', age: {}, number: {}},
  Object {name: {}, age: {}, number: 1},
  Object {name: {}, age: {}, number: 2},
   # /snip
}
```

这很好，但输出并没有向我们显示它们每一个的类型。要在 EdgeDB 的查询中引用对象自己的类型，你可以使用 `__type__`。只调用 `__type__` 只会给出一个 `uuid`，所以我们需要添加 `{name}` 来表明我们想要类型的名称。如果你想在查询中显示对象的类型，你可以访问这个字段，因为所有类型都有这个 `name` 字段。

```edgeql
SELECT Person {
  __type__: {
    name      # Name of the type inside module default
  },
  name, # Person.name
  [IS NPC].age,
  [IS Crewman].number,
};
```

从输出中选择前五个对象展示，现在看起来像这样：

```
{
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Woman 1', age: {}, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'The innkeeper', age: 30, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'Mina Murray', age: {}, number: {}},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 1},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 2},
}
```

这被称为 [多态查询（polymorphic query）](https://www.edgedb.com/docs/edgeql/overview/#ref-eql-polymorphic-queries)，并且是在你的架构中使用抽象类型的最佳理由之一.

## 超类型、子类型和泛型类型（Supertypes, subtypes, and generic types）

被其他类型扩展的类型的正式名称是 `supertype`（超类型）。扩展出的类型是它们的 `subtypes`（子类型）。因为继承一个类型会给你它的所有特性，所以`subtype IS supertype` 将返回 `{true}`。当然，`supertype IS subtype` 将返回 `{false}`，因为超类型不继承其子类型的特性。

在我们的架构中，这意味着 `SELECT PC IS Person` 返回 `{true}`，而 `SELECT Person IS PC` 将返回 `{true}` 或 `{false}`，具体取决于所选对象是否是`PC` .

想要通过查询显示这个不确定，只需使用可计算的公式`Person IS PC`  添加一个形状查询（shape query），EdgeDB 则会告诉你：

```edgeql
SELECT Person {
    name,
    is_PC := Person IS PC,
};
```

那么对于更简单的标量类型会怎么样？我们知道 EdgeDB 在整数、浮点数等不同类型方面是非常精确的，但是如果你只是想知道一个数字是否是整数呢？当然这会奏效，但不是十分令人满意：

```edgeql
WITH year := 1887,
SELECT year IS int16 OR year IS int32 OR year IS int64;
```

结果是：`{true}`.

但幸运的是，这些类型 [也都是从抽象类型扩展的](https://www.edgedb.com/docs/datamodel/abstract)，我们可以使用它们。这些抽象类型都以 `any` 开头，分别是 `anytype`，`anyscalar`，`anyenum`，`anytuple`，`anyint`，`anyfloat`，`anyreal`。唯一可能让你不确定的是 `anyreal`：它意味着任何实数，所以整数和浮点数，以及 `decimal` 类型。

因此，你可以将上述输入更改为 `SELECT 1887 IS anyint` 并获得 `{true}`。

## Multi 的使用（Multi in other places）

我们已经看到了很多次 `multi link`，你可能想知道 `multi` 是否也可以用在其他地方。答案是肯定的。比如 `multi property`，它与任何其他属性一样，但它可以有多个值。例如，我们的 `Castle` 类型有一个用于 `doors` 属性的 `array<int16>`：

```sdl
type Castle extending Place {
  property doors -> array<int16>;
}
```

像下面这样做可以达到同样效果：

```sdl
type Castle extending Place {
  multi property doors -> int16;
}
```

这样一来，你需要使用 `{}` 进行插入赋值，而不是方括号的数组：

```edgeql
INSERT Castle {
  name := 'Castle Dracula',
  doors := {6, 19, 10},
};
```

下一个问题当然是最好使用哪个：`multi property`、`array` 或通过链接的对象类型。答案是……视情况而定。但是这里有一些很好的经验法则可以帮助你决定选择哪个。

- `multi property` 
- `multi property` 与数组

  取决于你正在处理的数据有多大？当你有大量数据时，`multi property` 更有效，而数组则较慢。但是如果你处理的集合较小，那么数组比 `multi property` 更快。

  如果你想对单个元素使用索引和约束，那么你应该使用 `multi property`。我们将在第 16 章中学习索引，现在只需要知道它们是一种使查找更快的方法。

  如果顺序很重要，那么数组可能更好。因为更容易保持数组中项目的原始顺序。

- `multi property` 与对象

  在这里，我们先从 `multi property` 更好的两个方面开始，然后再说对象的好处。

  对象的第一个负面问题是：对象总是很大。还记得 `DESCRIBE TYPE as TEXT` 吗？让我们用它来看看我们之前创建的一个类型，`Castle` 类型：

  ```
  {
    'type default::Castle extending default::Place {
      required single link __type__ -> schema::Type {
        readonly := true;
      };
      optional single property doors -> array<std::int16>;
      required single property id -> std::uuid {
        readonly := true;
      };
      optional single property important_places -> array<std::str>;
      optional single property modern_name -> std::str;
      required single property name -> std::str;
  };
  ```

  你一定记得看到过 `readonly := true` 的类型，你创建的每个对象类型都会有它们。`__type__` 链接和 `id` 属性分别都是 16 字节。

  对象的第二个负面问题类似：对象更多是为计算机工作。EdgeDB 运行在 PostgreSQL 之上，指向对象的 `multi link` 需要额外的“连接（join）”（链接表 + 对象表），但 `multi property` 不需要。此外，“反向链接（backward link 或 reverse link）（将在第 14 章中看到）也需要更多的工作。

  好的，现在这里介绍一下经过比较后使用对象的两个好处。

  你是否有很多重复的属性值？如果是这样，那么在对象类型上使用 `constraint exclusive` 是更有效的方法。

  如果你需要每个对象具有多个值，则对象更容易迁移/变更（migrate）。

希望这些解释对你有所帮助。你可以看到你有很多选择，所以记住以上几点应该可以帮助你做出决定。大多数时候，你可能会知道自己想要哪个。

[→ 点击这里查看第 8 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何选择出所有 `Place` 和他们的名字，以及当它是个 `Castle` 时的属性 `door`？

2. 如何选择出 `Place`，用 `city_name` 显示当它是个 `City` 时的 `name`，并用 `country_name` 显示当它是个 `Country` 时的 `country_name`？

3. 基于上一题，如何做可以只显示属于 `City` 或 `Country` 类型的结果？

4. 你将如何显示所有没有 `lover` 的 `Person` 对象及其名称和类型名称？

5. 下面这个查询需要修复什么？提示：有两个地方是必须要修复的，还有一个地方可能应该更改以使其更具可读性。

   ```edgeql
   SELECT Place {
     __type__,
     name
     [IS Castle]doors
   };
   ```

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候认识苏厄德医生、亚瑟霍姆伍德和昆西莫里斯了……还有奇怪的伦菲尔德。_
