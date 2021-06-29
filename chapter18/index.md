---
tags: Complex Inserts, Schema Cleanup
---

# 第十八章 - 以牙还牙

> 范海辛（Van Helsing）是对的：米娜（Mina）与德古拉（Dracula）有关。他继续对米娜使用催眠术来了解德古拉在哪里以及他在做什么。乔纳森（Jonathan）对德古拉在伦敦的活动进行了大量的调查。他拜访了所有出售房子给德古拉的公司，以及一些给他搬运棺材的搬家公司。乔纳森越来越自信，从未停止寻找德古拉的工作。他们找到了德古拉在伦敦的另一所房子，里面有他所有的钱。知道他一定会来拿，他们便等他来……突然，德古拉跑进房子进行攻击。乔纳森用刀猛击德古拉，把他包里所有的钱都割破了。德古拉抓起一些掉下来的钱，从窗户跳了出去。德古拉冲他们吼道：“你们每个人都要后悔！你们以为你们让我无处可去；但其实我还有更多落脚地。我的报复才刚刚开始！”然后他就消失了。

这是一个很好的提醒，我们可能应该在游戏中引入“钱”的概念。书中的角色们去过英国、罗马尼亚和德国等国家，他们每个人都有自己的钱。“抽象类型”在这里似乎是一个不错的选择：我们应该创建一个 `abstract type Currency`，我们可以将其扩展为所有其他类型的货币。

现在，有一个困难是：在 1800 年代，货币体系比今天更复杂。例如，在英国，不是 100 便士兑换 1 磅，而是：

- 12 便士（最小单位的硬币）兑换一先令，
- 20 先令等于一磅，因此
- 每磅 240 便士。

（还有一个 _半便士铜币（halfpenny）_ 是一便士的一半，但让我们就不在我们的游戏中引入那么多细节了。）

为了说明这些，我们定义 `Currency` 具有三个属性：`major`、`minor` 和 `sub_minor`。每一个都会有一个金额，还会有一个用于换算的数字，再加上一个 `link owner -> Person`。所以 `Currency` 看起来像这样：

```sdl
abstract type Currency {
  required link owner -> Person;

  required property major -> str;
  required property major_amount -> int64 {
    default := 0;
    constraint min_value(0);
  }

  property minor -> str;
  property minor_amount -> int64 {
    default := 0;
    constraint min_value(0);
  }
  property minor_conversion -> int64;

  property sub_minor -> str;
  property sub_minor_amount -> int64 {
    default := 0;
    constraint min_value(0);
  }
  property sub_minor_conversion -> int64;
}
```

你会注意到只有属性 `major` 是 `required` 的，因为有些货币甚至没有美分之类的东西。在现代，包括日元、韩元等只是一个单一的货币单位加一个数字。

我们还给了它一个 `min_value(0)` 的约束，这样书中的角色们就不能透支他们的账户了。还有一些复杂的事情，比如信用和负货币，我们现在暂时先忽略。

然后来看一下我们的第一种货币：`Pound` 类型。属性 `minor` 称为 `'shilling'`，我们使用 `minor_conversion` 来说明获取以 1 磅所需的金额。`'pence'` 也一样。然后我们的角色可以收集各种硬币，但最终价值仍然可以很快变成英镑来表示。这是 `Pound` 类型：

```sdl
type Pound extending Currency {
  overloaded required property major {
    default := 'pound'
  }
  overloaded required property minor {
    default := 'shilling'
  }
  overloaded required property minor_conversion {
    default := 20
  }
  overloaded property sub_minor {
    default := 'pence'
  }
  overloaded property sub_minor_conversion {
    default := 240
  }
}
```

现在让我们给德古拉一些钱。我们给他 2500 英镑、50 先令和 200 便士。也许在 1887 年这是一大笔钱了。

```edgeql
INSERT Pound {
  owner := (SELECT Person filter .name = 'Count Dracula'),
  major_amount := 2500,
  minor_amount := 50,
  sub_minor_amount := 200
};
```

然后我们可以使用转换率以磅为单位来显示他拥有的总金额：

```edgeql
SELECT Currency {
  owner: {name},
  total := .major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)
};
```

他拥有这么多：

```
{default::Pound {owner: default::Vampire {name: 'Count Dracula'}, total: 2503.3333333333335}}
```

我们知道亚瑟（Arthur）（现在称为戈达尔明勋爵（Lord Godalming））有他需要的钱，但其他人我们不确定。让我们给他们中的一些人随机数量的钱，同时 `SELECT` 它以显示结果。对于随机数，我们将使用我们之前用于 `strength` 的方法：`round()` 一个 `random()` 数并乘以最大值。

最后，在显示总数时，我们将其转换为 `decimal` 类型。这样我们就可以将磅数显示为 555.76 而不是 555.76545256。为此，我们仍然使用 `round()` 函数，但使用最后一个签名：

```sdl
std::round(value: int64) -> float64
std::round(value: float64) -> float64
std::round(value: bigint) -> bigint
std::round(value: decimal) -> decimal
std::round(value: decimal, d: int64) -> decimal
```

该签名有一个额外的 `d: int64` 部分，用于表示我们想要给它的小数位数。

总之，它看起来像这样：

```edgeql
SELECT (
  FOR character IN {'Jonathan Harker', 'Mina Murray', 'The innkeeper', 'Emil Sinclair'}
  UNION (
    INSERT Pound {
      owner := (SELECT Person FILTER .name = character LIMIT 1),
      major_amount := (SELECT(round(random() * 500))),
      minor_amount := (SELECT(round(random() * 100))),
      sub_minor_amount := (SELECT(round(random() * 500)))
    }
  )
) {
  owner: {
    name
  },
  pounds := .major_amount,
  shillings := .minor_amount,
  pence := .sub_minor_amount,
  total_pounds := (
    SELECT (round(<decimal>(.major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)), 2))
  )
};
```

然后它会在下面的结果里给出我们要收集的钱，每个钱都有一个所有者：

```
{
  default::Pound {owner: default::NPC {name: 'Jonathan Harker'}, pounds: 54, shillings: 100, pence: 256, total_pounds: 60.07n},
  default::Pound {owner: default::NPC {name: 'Mina Murray'}, pounds: 360, shillings: 77, pence: 397, total_pounds: 365.50n},
  default::Pound {owner: default::NPC {name: 'The innkeeper'}, pounds: 87, shillings: 36, pence: 23, total_pounds: 88.90n},
  default::Pound {owner: default::PC {name: 'Emil Sinclair'}, pounds: 427, shillings: 19, pence: 88, total_pounds: 428.32n},
}
```

（如果你不想看到 `decimal` 类型最后的 `n`，只需将其转换为 `<float32>` 或 `<float64>`。）

你现在可能会注意到，关于如何显示金钱可能存在一些争论。它应该是一个 `Currency` 链接到所有者吗？或者它应该是一个 `Person` 链接到名为 `money` 的属性的？我们的方法对于现实游戏来说可能更简单，因为游戏中存在多种 `Currency` 类型。如果我们选择另一种方法，我们将有一个 `Person` 类型链接到每种类型的货币，并且大多数都为零。但是使用我们的方法，我们只需要在角色开始拥有某种货币时为其创造“成堆”的金钱。或者这些“堆”可能是钱包和袋子之类的东西，如果游戏中的角色可能会丢失它们，我们可以将 `required link owner -> Person;` 改为 `optional link owner -> Person;`。

当然，如果我们只有一种类型的钱，那么将它放在 `Person` 类型中会更简单。我们不会在我们的架构中这样做，但我们可以想象一下该如何做到这一点。如果游戏只在美国境内，那么在没有抽象的 `Currency` 类型的情况下这样做会更容易：

```sdl
type Dollar {
  required property dollars -> int64;
  required property cents -> int64;
  property total_money := .dollars + (.cents / 100)
}
```

顺便说一下，由于 `/ 100` 部分，`total_money` 类型将变成 `float64`。我们可以通过快速查询来确认这一点：

```edgeql
SELECT (100 + (55 / 100)) is float64;
```

结果是：`{true}`。

当我们进行插入并使用 `SELECT` 检查 `total_money` 属性时，我们可以看到相同的效果：

```edgeql
SELECT(
  INSERT Dollar {
    dollars := 100,
    cents := 55
  }
) {
  total_money
};
```

输出是：`{default::Dollar {total_money: 100.55}}`。完美！

这里并不是说我们在游戏中需要这种 `Dollar` 类型：在我们的架构中，它会是 `type Dollar extending Currency`。

最后一个注意事项：我们的 `total_money` 属性只是通过除以 100 创建的，因此它以小数有限的方式使用了 `float64`（这很好）。但是你要小心浮动，因为它们并不总是精确的，例如，如果我们需要除以 3，我们会得到类似 `100 / 3 = 33.33333333` 的结果……这对于实际货币来说不是很好。因此，在这种情况下，最好坚持使用整数。

## 清理架构（Cleaning up the schema）

我们已经接近本书的结尾了，可能应该开始清理一下我们的架构并插入一些内容。

首先，我们这里有两个插入，但我们可以只用一个插入。

```edgeql
INSERT City {
  name := 'Munich',
};

INSERT City {
  name := 'London',
};
```

我们将其更改为使用 `FOR` 循环的插入：

```edgeql
FOR city_name IN {'Munich', 'London'}
UNION (
  INSERT City {
    name := city_name
  }
);
```

然后，我们将对插入的四个 `Country` 对象（匈牙利、罗马尼亚、法国、斯洛伐克）执行相同的操作。如下是一个单个插入：

```edgeql
FOR country_name IN {'Hungary', 'Romania', 'France', 'Slovakia'}
UNION (
  INSERT Country {
    name := country_name
  }
);
```

其他 `City` 的插入有点不同：一些有 `modern_name`，一些有 `population`。在真正的游戏中，我们会以这种形式一次性将它们全部插入：

```edgeql
FOR city IN {
    ('City 1\'s name', 'City 1\'s modern name', 800),
    ('City 2\'s name', 'City 2\'s modern name', 900),
    ('City 3\'s name', 'City 3\'s modern name', 455),
  }
UNION (
  INSERT City {
    name := city.0,
    modern_name := city.1,
    population := city.2
  }
);
```

我们会对所有的 `NPC` 类型、它们的 `first_appearance` 数据等做同样的处理。但是我们在本教程中没有那么多的城市和角色要插入，所以我们还不需要那么系统。

我们还可以将 `Ship` 类型的插入件转换为一个单个插入。如下所示：

```edgeql
FOR n IN {1, 2, 3, 4, 5}
UNION (
  INSERT Crewman {
    number := n,
    first_appearance := cal::to_local_date(1887, 7, 6),
    last_appearance := cal::to_local_date(1887, 7, 16),
  }
);

INSERT Sailor {
  name := 'The Captain',
  rank := <Rank>Captain
};

INSERT Sailor {
  name := 'The First Mate',
  rank := <Rank>FirstMate
};

INSERT Sailor {
  name := 'The Second Mate',
  rank := <Rank>SecondMate
};

INSERT Sailor {
  name := 'The Cook',
  rank := <Rank>Cook
};

INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```

让我们把所有这些放在一起：

```edgeql
INSERT Ship {
  name := 'The Demeter',
  sailors := {
    (INSERT Sailor {
      name := 'The Captain',
      rank := <Rank>Captain
    }),
    (INSERT Sailor {
      name := 'The First Mate',
      rank := <Rank>FirstMate
    }),
    (INSERT Sailor {
      name := 'The Second Mate',
      rank := <Rank>SecondMate
    }),
    (INSERT Sailor {
      name := 'The Cook',
      rank := <Rank>Cook
    })
  },
  crew := (
    FOR n IN {1, 2, 3, 4, 5}
    UNION (
      INSERT Crewman {
        number := n,
        first_appearance := cal::to_local_date(1887, 7, 6),
        last_appearance := cal::to_local_date(1887, 7, 16),
      }
    )
  )
};
```

好多了！

[点击这里查看第 18 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 在德古拉时代，德国的货币使用的是金马克（Goldmark）。一金马克是 100 芬尼（Pfennig）。你会如何制作这种货币类型？

2. 尝试给这种类型添加两个注释（annotations）。其中一个称为 `description` 并说明 `One mark = 100 Pfennig`。另一个称为 `note`，并说明硬币的种类。

   [这里是硬币的种类](https://en.wikipedia.org/w/index.php?title=German_gold_mark&oldid=972733514#Base_metal_coins)：1, 2, 5, 10, 20, 25 芬尼（Pfennig）硬币。

3. 一个名叫戈德布兰（Godbrand）的吸血鬼刚刚袭击了一个村庄，将三个村民变成了 `MinorVampire`。你将如何一次插入涉及到的四个对象？

   下面他们的数据（姓名、出生日期（`first_appearance`）、变成 MinorVampire 的日期（`last_appearance`））：

   ```
   ('Fritz Frosch', '1850-01-15', '1887-09-11'),
   ('Levanta Sinyeva', '1862-02-24', '1887-09-11'),
   ('김훈', '1860-09-09', '1887-09-11'),
   ```

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _只有米娜可以告诉他们德古拉去了哪里。_
