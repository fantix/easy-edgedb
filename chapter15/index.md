---
tags: Expression On, Error Messages
---

# 第十五章 - 开始追捕吸血鬼

> 乔纳森（Jonathan）回来是件好事，但他仍然处于震惊之中。他不知道与德古拉（Dracula）的经历是否真实，他觉得自己可能疯了。直到后来他遇到了范海辛（Van Helsing），范海辛告诉他这一切都是真的。乔纳森听到后，再次变得坚强和自信。现在他们开始寻找德古拉。这时其他人得知苏厄德医生（Dr. Seward）的收容所对面的卡法克斯（Carfax）豪宅是德古拉买的。所以这就是为什么伦菲尔德（Renfield）受到如此强烈的影响……他们趁太阳升起时搜查房子，找到德古拉睡觉的箱子。他们将 Carfax 里的箱子全部摧毁了，但在伦敦仍然还有很多。如果他们不摧毁其他箱子，德古拉白天就可以在里面休息，每到晚上太阳落山时在出来恐吓伦敦。

## 更多抽象类型（More abstract types）

本章书里的人们学到了一些关于吸血鬼的知识：他们在白天需要睡在装有圣土的棺材里（用于装死人的盒子）。这就是为什么德古拉用德米特号（the Demeter）船带来了 50 个棺材。这对我们游戏的机制很重要，所以我们应该为此创建一个类型。如果我们考虑到：

- 世界上的每个地方要么有棺材要么没有，
- 有棺材（Has coffins）等于吸血鬼可以进入并恐吓人们，
- 如果一个地方有棺材，我们应该知道有多少。

这听起来像是抽象类型的一个好例子。如下所示：

```sdl
abstract type HasCoffins {
  required property coffins -> int16 {
    default := 0;
  }
}
```

大多数地方不会有专门供吸血鬼使用的棺材，所以默认为 0。`coffins` 属性是一个 `int16`，如果数字为 1 或更大，意味着吸血鬼可以留在附近。在我们游戏的机制中，我们可能会给吸血鬼一个距离有棺材大约 100 公里的活动半径。这是因为典型的吸血鬼作息表通常如下：

- 太阳下山后，精神焕发地从棺材里醒来，准备晚上 8 点离开，到处恐吓人类，
- 因为夜晚才刚刚开始，所以不用担心，于是离开安全的棺材去寻找受害者。可以使用以每小时 25 公里速度行驶的马车。
- 大约凌晨一两点时，开始感到紧张。大约 5 小时后太阳就会升起。有足够的时间回家吗？

所以晚上 8 点到凌晨 1 点之间是吸血鬼可以自由远离的时间，以 25 公里/小时的速度，他们可以在棺材周围获得大约 100 公里的活动半径。这个距离上，即使是最勇敢的吸血鬼也会在凌晨 2 点开始往家跑了。

对于更复杂的游戏，我们可以想象吸血鬼恐怖主义在冬天会更恶劣（因为活动半径可以扩大到约 150 公里），但我们不在这里对这些细节做深入的研究。

完成抽象类型 `HasCoffins` 的创建后，有很多类型需要 `extending`（扩展）自它。首先，我们可以让 `Place` 扩展它，这样就可以把它赋给所有其他的位置类型，比如 `City`、`OtherPlace`等等：

```sdl
abstract type Place extending HasCoffins {
  required property name -> str {
    constraint exclusive;
  };
  property modern_name -> str;
  property important_places -> array<str>;
}
```
船也是足够大，可以放置棺材的（毕竟德米特号承载了 50 个），所以我们还将扩展 `Ship`：
Ships are also big enough to have coffins (the Demeter had 50 of them, after all) so we'll extend for `Ship` as well:

```sdl
type Ship extending HasCoffins {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

如果我们愿意，我们现在可以创建一个快速函数来测试吸血鬼是否可以进入一个地方：

```sdl
function can_enter(person_name: str, place: HasCoffins) -> str
  using (
    with vampire := (SELECT Person filter .name = person_name LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

你可能注意到了这个函数中的 `person_name` 实际上只是一个用来选择 `Person` 的字符串。所以从技术上讲，它可能会有类似“乔纳森·哈克不能进入”的输出（但乔纳森是个人类）。尽管其中使用了 `LIMIT 1`，我们可以选择相信运用该函数的用户能够做到正确地使用它。但如果我们不能信任用户，这里有一些更健壮的选择：

- 重载函数以获得两个签名，每种类型的吸血鬼均有一个签名：

```sdl
function can_enter(vampire: Vampire, place: HasCoffins) -> str
function can_enter(vampire: MinorVampire, place: HasCoffins) -> str
```

- 创建一个抽象类型（如 `type IsVampire`）并使其扩展自 `Vampire` 和 `MinorVampire`。然后 `can_enter` 函数可以有签名：`function can_enter(vampire: IsVampire, place: HasCoffins) -> str`。

重载函数可能是更简单的选择，因为我们不需要进行显式迁移（explicit migration）。

> TODO: translate after fixing the original content.

One other area where you need to trust the user of the function is seen in the return type, which is just `-> str`. Beyond just returning a string, this return type also means that the function won't be called if the input is empty. So what if you want it to be called anyway? If you want it to be called no matter what, you can change the return type to `-> OPTIONAL str`. [The documentation](https://www.edgedb.com/docs/edgeql/overview#optional) explains it like this: `the function is called normally when the corresponding argument is empty`. And: `A notable example of a function that gets called on empty input is the coalescing operator.`

Interesting! You'll remember the coalescing operator `??` that we first saw in Chapter 12. And when we look at [its signature](https://www.edgedb.com/docs/edgeql/funcops/set/#operator::COALESCE), you can see the `OPTIONAL` in there:

`OPTIONAL anytype ?? SET OF anytype -> SET OF anytype`

因此，这些是关于如何设置功能的一些想法，具体取决于你认为用户可能会如何使用它们。

现在，让我们来给伦敦（London）放置一些棺材。根据小说，我们的英雄们在那天晚上摧毁了卡法克斯（Carfax）里的 29 具棺材，也就是说还有 21 具在伦敦。

```edgeql
UPDATE City filter .name = 'London'
SET {
  coffins := 21
};
```

现在我们终于可以调用我们的函数，看看它是否有效：

```edgeql
SELECT can_enter('Count Dracula', (SELECT City filter .name = 'London'));
```

得到：`{'Count Dracula can enter.'}`.

还可以有一些其他可能的改进 `can_enter()` 的想法，比如：

- 将属性 `name` 从 `Place` 和 `Ship` 移到 `HasCoffins`。然后用户可以只是输入一个字符串（名字）。函数里将通过它 `SELECT` 到对应的类型，然后显示其名称，给出类似 `{'Count Dracula can enter.'}` 的结果。
- 给函数加一个日期类型的输入，以便我们可以先检查这个吸血鬼是否死了。例如，如果我们输入一个露西死后的日期，它只会显示类似 `vampire.name ++ ' is already dead on ' ++ <str>.date ++ ' and cannot enter ' ++ city.name`。

## 更多约束（More constraints）

让我们看看更多的约束限制。我们已经见到过了 `exclusive` 和 `max_value`，这里还有 [一些其他的](https://www.edgedb.com/docs/datamodel/constraints) 我们也可以使用。

有一种叫做 `max_len_value` 的方法可以确保字符串不会超过特定长度。这可能可以用于我们在许多章节之前创建的 `PC` 类型。我们只用过一次作为测试，因为我们还没有任何玩家。我们一直是参考这本小说（《德古拉》）来为我们想象的游戏创建 `NPC` 的数据库。`NPC` 不需要这个约束，因为他们的名字已经确定了，但是 `max_len_value()` 可以用于 `PC` 以确保玩家不会选择疯狂（太长）的名字。因此，我们可以将 `PC` 更改为如下所示：

```sdl
type PC extending Person {
  required property transport -> Transport;
  overloaded required property name -> str {
    constraint max_len_value(30);
  }
}
```

然后当我们尝试插入一个名字太长的 `PC` 时，会被拒绝，并得到错误提示：`ERROR: ConstraintViolationError: name must be no longer than 30 characters`。

另一个方便的约束叫做 `one_of`，有点像枚举。在我们的架构中，我们可以使用到它的一个地方是我们的 `Person` 类型中的 `property title -> str;`。你一定还记得我们添加这个属性，以防我们想从各个部分（名字、姓氏、头衔、学位……）组合生成名称。这种约束可以确保人们不会给自己编造头衔：

```sdl
property title -> str {
  constraint one_of('Mr.', 'Mrs.', 'Ms.', 'Lord')
}
```

对我们来说，添加一个 `one_of` 约束可能不太值得，因为整本书中可能有太多的头衔（伯爵、先生，等等）。

另一个你可以想到的使用 `one_of` 的地方是“月份”，因为这本小书的时间只涉及了同年的五月到十月。如果我们有一个生成日期的对象类型，那么你可以在其中包含该约束：

```sdl
property month -> int64 {
  constraint one_of(5, 6, 7, 8, 9, 10)
}
```

但这将取决于游戏将如何设定。

现在让我们学习一下可能是 EdgeDB 中最有趣的约束：

## 最灵活的约束（expression on: the most flexible constraint）

还一种特别灵活的约束，叫做 [`expression on`](https://www.edgedb.com/docs/datamodel/constraints#constraint::std::expression)，它允许我们添加任何我们想要的表达式（约束）。在 `expression on` 之后（的小括号中）添加必须为“真”的表达式即可。

假设我们稍后会出于某种原因需要一个 `Lord` 类型，并且所有 `Lord` 类型的对象名称中都必须包含“Lord”这个词。我们可以约束该类型以确保始总是如此。为此，我们使用一个名为 [contains()](https://www.edgedb.com/docs/edgeql/funcops/generic#function::std::contains) 的函数，如下所示：

```sdl
std::contains(haystack: str, needle: str) -> bool
```

如果 `haystack`（一个字符串）包含 `needle`（通常是一个较短的字符串），它返回 `{true}`。

我们可以像下面这样用 `expression on` 和 `contains()` 来编写这个约束：

```sdl
type Lord extending Person {
  constraint expression on (
    contains(__subject__.name, 'Lord') = true
  );
}
```

这里的 `__subject__` 是指类型本身。

现在，当我们尝试插入一个名字中没有“Lord”的 `Lord` 时，将无法成功：

```edgeql
INSERT Lord {
  name := 'Billy'
  # Other stuff..
};
```

但是如果 `name` 是“Lord Billy”（或“Lord” + 任何东西），它就会正常工作。

在此期间，让我们练习一下 `SELECT` 和 `INSERT` 的同时执行，以便我们立即看到 `INSERT` 的结果。我们把 `Billy` 改为 `Lord Billy`，且比利勋爵（Lord Billy）因为有很多财富，到访过我们数据库中的所有地方。

```edgeql
SELECT (
  INSERT Lord {
    name := 'Lord Billy',
    places_visited := (SELECT Place),
  }
) {
  name,
  places_visited: {
    name
  }
};
```

`.name` 包含了子字符串 `Lord`，因此它能正常工作：

```
{
  default::Lord {
    name: 'Lord Billy',
    places_visited: {
      default::Castle {name: 'Castle Dracula'},
      default::City {name: 'Whitby'},
      default::City {name: 'Munich'},
      default::City {name: 'Buda-Pesth'},
      default::City {name: 'Bistritz'},
      default::City {name: 'London'},
      default::Country {name: 'Romania'},
      default::Country {name: 'Slovakia'},
    },
  },
}
```

## 自定义错误信息（Setting your own error messages）

由于 `expression on` 非常的灵活，因此你可以用任何你能想到的方式来使用它。但没法确定用户可以了解到这个约束的具体内容 —— 即没有对应的消息通知。同时，我们现在得到的系统自动生成的错误信息（如下所示）并无法带来什么有用的帮助：

`ERROR: ConstraintViolationError: invalid Lord`

所以没有办法告知这里的问题是 `name` 里面需要包含 `Lord`。幸运的是，约束允许你通过使用 `errmessage` 来设定自己的错误提示信息，比如：`errmessage := "All lords need 'Lord' in their name."`

现在，错误提示变成了：

`ERROR: ConstraintViolationError: All lords need 'Lord' in their name.`

下面是 `Lord` 类型现在的样子：

```sdl
type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord') = true) {
    errmessage := "All lords need \'Lord\' in their name";
  };
};
```

## Links in two directions

回到第 6 章，我们从 `MinorVampire` 中删除了 `link master`，因为 `Vampire` 已经有了 `link slave` 到 `MinorVampire` 类型。一个原因是复杂性，另一个原因是 `DELETE` 变得不可行，因为它们彼此依赖。但现在我们知道如何使用反向链接了，如果我们想，我们随时可以把 `master` 放回 `MinorVampire`。

（注意：实际上，我们不会在这里改变 `MinorVampire` 类型，因为我们已经知道如何通过反向查找访问 `Vampire`，这里是为了说明如何做到的）

首先，这是目前的 `MinorVampire` 类型：

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
}
```

> TODO: translate after fixing the original content.

To add the `master` link again, one way to start would be with a property called `master_name` that is just a string. Then we can use this in a reverse search to link to the `Vampire` type if the name matches. It's a single link, so we'll add `LIMIT 1` (it won't work otherwise). Here is what the type would look like now:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
  link master := (SELECT .<slaves[IS Vampire] LIMIT 1);
  required single property master_name -> str;
};
```

But then again, if we want this `master_name` shortcut we can now just use the `master` link to do it. Let's change it from `required single property master_name -> str` to `property master_name := .master.name`:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
  link master := (SELECT .<slaves[IS Vampire] LIMIT 1);
  property master_name := .master.name;
};
```

现在让我们来测试一下。我们来创建一个名为“Kain”的吸血鬼，他有两个 `MinorVampire` 奴隶，分别叫做“Billy”和“Bob”。

```edgeql
INSERT Vampire {
  name := 'Kain',
  slaves := {
    (INSERT MinorVampire {
      name := 'Billy',
    }),
    (INSERT MinorVampire {
      name := 'Bob',
    })
  }
};
```

Now if the `MinorVampire` type works as it should, we should be able to see Kain via `link master` inside `MinorVampire` and we won't have to do a reverse lookup. Let's check:

```edgeql
SELECT MinorVampire {
  name,
  master_name,
  master: {
    name
  }
} FILTER .name IN {'Billy', 'Bob'};
```

And the result:

```
{
  default::MinorVampire {
    name: 'Billy',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
  default::MinorVampire {
    name: 'Bob',
    master_name: 'Kain',
    master: default::Vampire {name: 'Kain'},
  },
}
```

完美！所有的信息都在这里了。

[点击这里查看第 15 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何创建一个名为“Horse”的类型，且它的属性 `required property name -> str` 只能是“Horse”？

2. 如何让用户们能了解到 `name` 需要被叫做“Horse”？如何提示他们？

3. 如何确保 `NPC` 类型的 `name` 长度始终在 5 到 30 个字符之间？

   先用`expression on`试试。

4. 如何创建一个名为 `display_coffins` 的函数来显示出所有棺材数量大于 0 的 `HasCoffins` 类型的对象？

5. 如何在不触及架构（schema）的情况下创建上一题中的函数？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _伦菲尔德能帮上忙吗？_
