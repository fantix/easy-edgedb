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

A lot of characters are starting to die now, so let's think about that. We could come up with a method to see who is alive and who is dead, depending on a `cal::local_date`. First let's take a look at the `People` objects we have so far. We can easily count them with `SELECT count(Person)`, which gives `{23}`.

There is also a function called [`enumerate()`](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::enumerate) that gives tuples of the index and the set that we give it. We'll use this to compare to our `count()` function to make sure that our number is right.

First a simple example of how to use `enumerate()`:

```edgeql
WITH three_things := {'first', 'second', 'third'},
SELECT enumerate(three_things);
```

The output is:

```
{(0, 'first'), (1, 'second'), (2, 'third')}
```

So now let's use it with `SELECT enumerate(Person.name);` to make sure that we have 23 results. The last index should be 22:

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

There are only 18? Oh, that's right: the `Crewman` objects don't have a name so they don't show up. How can we get them in the query? We could of course try something fancy like this:

```edgeql
WITH
  a := array_agg((SELECT enumerate(Person.name))),
  b:= array_agg((SELECT enumerate(Crewman.number))),
SELECT (a, b);
```

(`array_agg()` is to avoid multiplying sets by sets, as we saw in Chapter 12)

But the result is less than satisfying:

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

The `Crewman` types are now just numbers, which doesn't look good. Let's give up on fancy queries and just update them with names based on the numbers instead. This will be easy:

```edgeql
UPDATE Crewman
SET {
  name := 'Crewman ' ++ <str>.number
};
```

By the way, we don't have any more `Crewman` types to add but if we did, then we could just change the schema to this to avoid needing `UPDATE`:

```
type Crewman extending HasNumber, Person {
  overloaded property name := 'Crewman ' ++ <str>.number; #this part is new
}
```

So now that everyone has a name, let's use that to see if they are dead or not. The logic is simple: we input a `cal::local_date`, and if it's greater than the date for `last_appearance` then the character is dead.

```edgeql
WITH p := (SELECT Person),
     date := <cal::local_date>'1887-08-16',
SELECT(p.name, p.last_appearance, 'Dead on ' ++ <str>date ++ '? ' ++ <str>(date > p.last_appearance));
```

Here is the output:

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

We could of course turn this into a function if we use it enough.

## Reverse links

Finally, let's look at how to follow links in reverse direction, one of EdgeDB's most powerful and useful features. Learning to use it can take a bit of effort, but it's well worth it.

We know how to get Count Dracula's `slaves` by name with something like this:

```edgeql
SELECT Vampire {
  name,
  slaves: {
    name
  }
};
```

That shows us the following:

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

But what if we are doing the opposite? Namely, starting from `SELECT MinorVampire` and wanting to access the `Vampire` type connected to it. Because right now, we can only bring up the properties that belong to the `MinorVampire` and `Person` type. Consider the following:

```edgeql
SELECT MinorVampire {
  name,
  # master... how do we get this?
  # There's no link to Vampire inside MinorVampire...
}
```

Since there's no `link master -> Vampire`, how do we go backwards to see the `Vampire` type that links to it?

This is where reverse links come in, where we use `.<` instead of `.` and specify the type we are looking for: `[IS Vampire]`.

First let's move out of our `MinorVampire` query and just look at how `.<` works. Here is one example:

```edgeql
SELECT MinorVampire.<slaves[IS Vampire] {
  name,
  age
};
```

Because it goes in reverse order, it is selecting `Vampire` that has `.slaves` that are of type `MinorVampire`.

You can think of `MinorVampire.<slaves[IS Vampire] {name, age}` as "Select the name and age of the Vampire type with slaves that are of type MinorVampire" - from right to left.

Here is the output:

```
{default::Vampire {name: 'Count Dracula', age: 800}}
```

So far that's the same as just `SELECT Vampire: {name, age}`. But it becomes very useful in our query before, where we wanted to access multiple types. Now we can select all the `MinorVampire` types and their master:

```edgeql
SELECT MinorVampire {
  name,
  master := .<slaves[IS Vampire] {name},
};
```

You could read `.<slaves[IS Vampire] {name}` as "the name of the `Vampire` type that links back to `MinorVampire` through `.slaves`".

Here is the output:

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

1. How would you display just the numbers for all the `Person` types? e.g. if there are 20 of them, displaying `1, 2, 3..., 18, 19, 20`.

2. Using reverse lookup, how would you display 1) all the `Place` types (plus their names) that have an `o` in the name and 2) the names of the people that visited them?

3. Using reverse lookup, how would you display all the Person types that will later become `MinorVampire`s?

   Hint: Remember, `MinorVampire` has a link back to the vampire's former self.

4. How would you give the `MinorVampire` type an annotation called `note` that says `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`?

5. How would you see this `note` annotation for `MinorVampire` in a query?

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _是时候报仇了。_
