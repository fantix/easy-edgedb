# Chapter 14 Questions and Answers

#### 1. 如何仅显示所有 `Person` 对象的编号？比如，如果有 20 个，则显示 `1, 2, 3..., 18, 19, 20`。

一种方法是使用 `enumerate()`。如果我们只是从 0 开始显示，这就容易。enumerate() 会给出一个包含两个项目的元组。第一个则是 `int64` 类型的索引，所以我们选择它：

```edgeql
SELECT enumerate(Person).0;
```

将显示出：`{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19}`

警告：选择 `.1` 会产生错误，因为这是对象的其余部分，且没有办法正确显示它。但是显示它的单个属性没问题，例如在下面的示例中：

```edgeql
SELECT enumerate(Person.strength).1;
```

回到问题，如果要显示从 1 开始的数字，我们只需在使用 `enumerate` 时将其加 1：

```edgeql
SELECT enumerate(Person).0 + 1
```

#### 2. 使用反向查找，你将如何显示 1）所有名称中带有 `o` 的 `Place` 的对象（及他们的名字）；2）访问过这些地方的人的名字？

如果你一步一步开始，这并不太难，首先用一个过滤器来获取所有名字满足条件的 `Place` 对象：

```edgeql
SELECT Place {
  name
} FILTER .name LIKE '%o%';
```

这是结果：

```edgeql
{
  default::Country {name: 'Romania'},
  default::Country {name: 'Slovakia'},
  default::City {name: 'London'},
}
```

现在我们将反向查找添加到同一个查询，并调用可计算的 `visitors`：

```edgeql
SELECT Place {
  name,
  visitors := .<places_visited[IS Person].name
} FILTER .name LIKE '%o%';
```

现在我们可以看到谁到访过这些地方：

```
{
  default::Country {name: 'Romania', visitors: {}},
  default::Country {name: 'Slovakia', visitors: {}},
  default::City {
    name: 'London',
    visitors: {
      'Lucy Westenra',
      'The innkeeper',
      'Mina Murray',
      'John Seward',
      'Quincey Morris',
      'Arthur Holmwood',
      'Renfield',
      'Abraham Van Helsing',
      'Emil Sinclair',
    },
  },
}
```

伦敦作为访问量最大的地方的明显胜出！如果你想，你也可以添加一个 `visitor_numbers := count(.<places_visited[IS Person].name)` 来获取访问者的数量。

#### 3. 使用反向查找，你将如何显示所有后来成为了 `MinorVampire` 的 `Person` 对象？

我们可以再次使用可计算公式来做到这一点，我们将其称为 `later_vampire`。然后我们使用反向查找链接回 `MinorVampire`，即通过属性 `former_self` 链接到 `Person` 的 `MinorVampire`：

```edgeql
SELECT Person {
  name,
  later_vampire := .<former_self[IS MinorVampire].name
} FILTER exists .later_vampire;
```

这只是给了我们露西：

`{default::NPC {name: 'Lucy Westenra', later_vampire: {'Lucy Westenra'}}}`

还不错，但我们可能可以做得更好 —— 这里的 `later_vampire` 没有告诉我们它的类型的信息。让我们为其添加一些类型信息：
This is not bad, but we can probably do better - `later_vampire` here isn't telling us anything about the type. Let's add some type info:

```edgeql
SELECT Person {
  name,
  later_vampire := .<former_self[IS MinorVampire] {
    name,
    __type__: {
      name
    }
  }
} FILTER exists .later_vampire;
```

现在我们可以看到 `later_vampire` 的类型是 `MinorVampire` 而不是仅仅显示一个字符串：

```
{
  default::NPC {
    name: 'Lucy Westenra',
    later_vampire: {
      default::MinorVampire {
        name: 'Lucy Westenra',
        __type__: schema::ObjectType {name: 'default::MinorVampire'},
      },
    },
  },
}
```

#### 4. 如何给 `MinorVampire` 类型一个名为 `note` 的注解，说明 `'first_appearance for MinorVampire should always match last_appearance for its matching NPC type'`？

首先，你得创建这个注释，因为它还不存在：

```sdl
abstract annotation note;
```

然后，您只需将其放入 `MinorVampire` 类型即可完成！

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
  annotation note := 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type';
}
```

#### 5. 如何在查询中看到 `MinorVampire` 的 `note` 注释？

如下所示：

```edgeql
SELECT (INTROSPECT MinorVampire) {
  name,
  annotations: {
    name,
    @value
  }
};
```

这是输出：

```
{
  schema::ObjectType {
    name: 'default::MinorVampire',
    annotations: {
      schema::Annotation {
        name: 'std::description',
        @value: 'first_appearance for MinorVampire should always match last_appearance for its matching NPC type',
      },
    },
  },
}
```

当然，`@value` 本身会显示所有注释的字符串，但这里我们只有一个。
