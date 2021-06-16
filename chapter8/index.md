---
tags: Multiple Inheritance, Polymorphism
---

# 第八章 - 德古拉跨乘船前往英国

本章我们终于离开了德古拉城堡。这是本章中发生的事情：

> 一艘船从保加利亚（Bulgaria）的瓦尔纳市（Varna）出发，驶入黑海。它有一个**船长（captain）、大副（first mate）、二副（second mate）、厨师（cook）** 和 **五名船员（five crew）**。德古拉也在床上，但没人知道他在。每天晚上德古拉都会离开他的棺材，且每天晚上都会有一个人消失。船上的人感到害怕，但不知道发生了什么或该做什么。其中一个人说他看到一个奇怪的人在甲板上走来走去，但其他人不相信他。终于是船抵达英国（England）惠特比市（Whitby）的最后一天了，只剩下船长一个人了 —— 其他所有人都消失了。船长现在知道真相了。他将双手绑在方向盘上，这样即使德古拉找到了他，船也能继续直行。第二天，惠特比（Whitby）的人们看到一艘船搁浅了，一只狼从上面跳下来并跑到了岸上 —— 这是德古拉的狼化身，但人们并不知道。人们发现死去的船长被绑在方向盘上，手里拿着一个笔记本，于是开始阅读里面的故事。

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

So now we have quite a few types that extend the `Person` type, many with their own properties. The `Crewman` type has a property `number`, while the `NPC` type has a property called `age`.

But this gives us a problem if we want to query them all at the same time. They all extend `Person`, but `Person` doesn't have all of their links and properties. So this query won't work:

```edgeql
SELECT Person {
  name,
  age,
  number,
};
```

The error is `ERROR: InvalidReferenceError: object type 'default::Person' has no link or property 'age'`.

Luckily there is an easy fix for this: we can use `IS` inside square brackets to specify the type. Here's how it works:

- `.name`: this stays the same, because `Person` has this property
- `.age`: this belongs to the `NPC` type, so change it to `[IS NPC].age`
- `.number`: this belongs to the `Crewman` type, so change it to `[IS Crewman].number`

Now it will work:

```edgeql
SELECT Person {
  name,
  [IS NPC].age,
  [IS Crewman].number,
};
```

The output is now quite large, so here's just a part of it. You'll notice that types that don't have a property or a link will return an empty set: `{}`.

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

This is pretty good, but the output doesn't show us the type for each of them. To refer to an object's own type in a query in EdgeDB you can use `__type__`. Calling just `__type__` will just give a `uuid` though, so we need to add `{name}` to indicate that we want the name of the type. All types have this `name` field that you can access if you want to show the object type in a query.

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

Choosing the five objects from before from the output, it now looks like this:

```
{
  Object {__type__: Object {name: 'default::MinorVampire'}, name: 'Woman 1', age: {}, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'The innkeeper', age: 30, number: {}},
  Object {__type__: Object {name: 'default::NPC'}, name: 'Mina Murray', age: {}, number: {}},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 1},
  Object {__type__: Object {name: 'default::Crewman'}, name: {}, age: {}, number: 2},
}
```

This is officially called a [polymorphic query](https://www.edgedb.com/docs/edgeql/overview/#ref-eql-polymorphic-queries), and is one of the best reasons to use abstract types in your schema.

## 超类型、子类型和泛型类型（Supertypes, subtypes, and generic types）

The official name for a type that gets extended by another type is a `supertype` (meaning 'above type'). The types that extend them are their `subtypes` ('below types'). Because inheriting a type gives you all of its features, `subtype IS supertype` will return `{true}`. And of course, `supertype IS subtype` returns `{false}` because supertypes do not inherit the features of their subtypes.

In our schema, that means that `SELECT PC IS Person` returns `{true}`, while `SELECT Person IS PC` will return `{true}` or `{false}` depending on whether the object is a `PC`.

To make a query that will show this, just add a shape query with the computable `Person IS PC` and EdgeDB will tell you:

```edgeql
SELECT Person {
    name,
    is_PC := Person IS PC,
};
```

Now how about the simpler scalar types? We know that EdgeDB is very precise in having different types for integers, floats and so on, but what if you just want to know if a number is an integer for example? Of course this will work, but it's not very satisfying:

```edgeql
WITH year := 1887,
SELECT year IS int16 OR year IS int32 OR year IS int64;
```

Output: `{true}`.

But fortunately these types all [extend from abstract types too](https://www.edgedb.com/docs/datamodel/abstract), and we can use them. These abstract types all start with `any`, and are: `anytype`, `anyscalar`, `anyenum`, `anytuple`, `anyint`, `anyfloat`, `anyreal`. The only one that might make you pause is `anyreal`: this one means any real number, so both integers and floats, plus the `decimal` type.

So with that you can change the above input to `SELECT 1887 IS anyint` and get `{true}`.

## Multi 的使用（Multi in other places）

We've seen `multi link` quite a bit already, and you might be wondering if `multi` can appear in other places too. The answer is yes. A `multi property` is like any other property, except that it can have more than one value. For example, our `Castle` type has an `array<int16>` for the `doors` property:

```sdl
type Castle extending Place {
  property doors -> array<int16>;
}
```

But it could do something similar like this:

```sdl
type Castle extending Place {
  multi property doors -> int16;
}
```

With that, you would insert using `{}` instead of square brackets for an array:

```edgeql
INSERT Castle {
  name := 'Castle Dracula',
  doors := {6, 19, 10},
};
```

The next question of course is which is best to use: `multi property`, `array`, or an object type via a link. The answer is...it depends. But here are some good rules of thumb to help you decide which to choose.

- `multi property` vs. arrays:

  How large is the data you are working with? A `multi property` is more efficient when you have a lot of data, while arrays are slower. But if you have small sets, then arrays are faster than `multi property`.

  If you want to use indexes and constraints on individual elements, then you should use a `multi property`. We'll look at indexes in Chapter 16, but for now just know that they are a way of making lookups faster.

  If order is important, than an array may be better. It's easier to keep the original order of items in an array.

- `multi property` vs. objects

  Here we'll start with two areas where `multi property` is better, and then two areas where objects are better.

  First negative for objects: objects are always larger, and here's why. Remember `DESCRIBE TYPE as TEXT`? Let's look at one of our types with that again. Here's the `Castle` type:

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

  You'll remember seeing the `readonly := true` types, which are created for each object type you make. The `__type__` link and `id` property together always make up 32 bytes.

  The second negative for objects is similar: underneath, they are more work for the computer. EdgeDB runs on top of PostgreSQL, and a `multi link` to an object needs an extra "join" (a link table + object table), but a multi property only has one. Also, a "backward link" or "reverse link" (you'll see those in Chapter 14) takes more work as well.

  Okay, now here are two positives for objects in comparison.

  Do you have a lot of duplication in property values? If so, then using `constraint exclusive` on an object type is the more efficient way to do it.

  Objects are easier to migrate if you need to have more than one value with each.

So hopefully that explanation should help. You can see that you have a lot of choice, so remembering the points above should help you make a decision. Most of the time, you'll probably have a sense for which one you want.

[这里是第八章中到目前为止的所有代码。](code.md)

<!-- quiz-start -->

## 章节小练习

1. How would you select all the `Place` types and their names, plus the `door` property if it's a `Castle`?

2. How would you select `Place` types with `city_name` for `name` if it's a `City` and `country_name` for `name` if it's a `Country`?

3. How would you do the same but only showing the results of `City` and `Country` types?

4. How would you display all the `Person` types that don't have `lover`s, with their names and their type names?

5. What needs to be fixed in this query? Hint: two things definitely need to be fixed, while one more should probably be changed to make it more readable.

   ```edgeql
   SELECT Place {
     __type__,
     name
     [IS Castle]doors
   };
   ```

[可以在这里查看答案。](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候认识苏厄德博士、亚瑟霍姆伍德和昆西莫里斯了……还有奇怪的伦菲尔德。_
