---
tags: Aliases, Named Tuples
---

# 第十七章 - 可怜的伦菲尔德，可怜的米娜

> 上一章里，苏厄德医生（Dr. Seward）和范海辛医生（Dr. Van Helsing）想放伦菲尔德（Renfield）出去，但他们无法信任他。但事实证明，伦菲尔德说的是实话！那天晚上，德古拉（Dracula）发现他们正在摧毁他的棺材，于是决定袭击米娜（Mina）。他成功了，现在米娜正在慢慢变成一只吸血鬼。虽然她仍然是人类，但她现在与德古拉有了联系。

> 大家发现伦菲尔德躺在血泊中，奄奄一息。伦菲尔德很抱歉，告诉了他们真相。他和德古拉有联系，以为德古拉也会帮助他变成吸血鬼，所以他让德古拉进了屋子。但是一进屋，德古拉就没有理他，而是径直走向了米娜的房间。伦菲尔德攻击德古拉试图阻止他伤害米娜，但德古拉太强壮了，打不过。

> 不过，米娜并没有放弃，她有一个好主意。如果她现在与德古拉有联系，如果范海辛（Van Helsing）对她使用催眠术会发生什么？那能行吗？范海辛拿出怀表对米娜说：“请专心看这只表。你正在感到困倦……你有什么感觉？想想那个袭击你的人，试着去感受他在哪里……” 

## 命名元组（Named tuples）

还记得我们创建的 `fight()` 函数吗？它被重载后可以接受 `(Person, Person)` 或 `(str, Person)` 作为输入。现在让我们将德古拉（Dracula）和伦菲尔德（Renfield）输入进去：

```edgeql
WITH
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),
  renfield := (SELECT Person FILTER .name = 'Renfield'),
SELECT fight(dracula, renfield);
```

毫无疑问，结果当然会是 `{'Count Dracula wins!'}`。

执行同样查询的另一种方法是使用单个元组。然后我们可以将他们输入函数，其中德古拉为 `.0`，伦菲尔德为 `.1`。

```edgeql
WITH fighters := (
    (SELECT Person FILTER .name = 'Count Dracula'),
    (SELECT Person FILTER .name = 'Renfield')
  ),
SELECT fight(fighters.0, fighters.1);
```

看起来还不错，但有一种方法可以使它更清楚：我们可以为元组中的项目命名，来替代使用 `.0` 和 `.1`。它看起来像一个普通的可计算公式，使用`:=`：

```edgeql
WITH fighters := (
    dracula := (SELECT Person FILTER .name = 'Count Dracula'),
    renfield := (SELECT Person FILTER .name = 'Renfield')
  ),
SELECT fight(fighters.dracula, fighters.renfield);
```

下面是命名元组的另一个示例：

```edgeql
WITH minor_vampires := (
    women := (SELECT MinorVampire FILTER .name LIKE '%Woman%'),
    lucy := (SELECT MinorVampire FILTER .name LIKE '%Lucy%')
  ),
SELECT (minor_vampires.women.name, minor_vampires.lucy.name);
```

输出是：

```
{('Woman 1', 'Lucy Westenra'), ('Woman 2', 'Lucy Westenra'), ('Woman 3', 'Lucy Westenra')}
```

伦菲尔德已经不在人世了，所以我们需要使用 `UPDATE` 给他一个 `last_appearance`。让我们再做一个有趣的 —— `SELECT` 我们刚刚更新的并显示其信息：

```edgeql
SELECT ( # Put the whole update inside
  UPDATE NPC filter .name = 'Renfield'
  SET {
    last_appearance := <cal::local_date>'1887-10-03'
  }
) # then use it to call up name and last_appearance
{
  name,
  last_appearance
};
```

结果是：`{default::NPC {name: 'Renfield', last_appearance: <cal::local_date>'1887-10-03'}}`

最后要提的是：在元组中命名一个项目对项目本身没有任何影响。所以：

```edgeql
SELECT ('Lucy Westenra', 'Renfield') = (character1 := 'Lucy Westenra', character2 := 'Renfield');
```

返回 `{true}`。

## 将抽象类型放在一起（Putting abstract types together）

哪里有吸血鬼，哪里就有吸血鬼猎人。有时猎人会摧毁棺材，有时吸血鬼会建造更多。因此，是时候创建一个函数 `change_coffins()` 用来快速更改某个地方的棺材数量了。例如，使用此函数，我们可以编写类似 `change_coffins('London', -13)` 之类的来将伦敦的棺材数减少 13。但现在的问题是：

- `HasCoffins` 类型是一个抽象类型，具有一个属性：`coffins`，
- 可以放置棺材的地方是 `Place` 及其扩展出的所有类型，再加上 `Ship`，
- 最好的过滤方式是通过 `.name`，但是 `HasCoffins` 没有这个属性。

所以也许我们可以把这个类型转换成一个叫做 `HasNameAndCoffins` 的类型，然后把 `name` 和 `coffins` 两个属性放在里面。这没有什么问题，因为在我们的游戏中每个地方都需要一个名字和一些可能存在的棺材。请记住，0 棺材意味着吸血鬼不能在一个地方呆很长时间：只能在夜间、太阳升起前快速行动。

下面是具有新属性 `name` 的类型。我们会给它两个约束：`exclusive` 和 `max_len_value` 以防止重名及名称太长。

```sdl
abstract type HasNameAndCoffins {
  required property coffins -> int16 {
    default := 0;
  }
  required property name -> str {
    constraint exclusive;
    constraint max_len_value(30);
  }
}
```

所以现在我们可以更改我们的 `Ship` 类型了（注意我们删除了 `name`）

```sdl
type Ship extending HasNameAndCoffins {
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

还有 `Place` 类型。现在简单多了。

```sdl
abstract type Place extending HasNameAndCoffins {
  property modern_name -> str;
  property important_places -> array<str>;
}
```

最后，我们可以更改我们的 `can_enter()` 函数。之前它需要一个 `HasCoffins` 类型作为输入之一：

```sdl
function can_enter(person_name: str, place: HasCoffins) -> str
  using (
    with vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
    SELECT vampire.name ++ ' can enter.' IF place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

但是现在 `HasNameAndCoffins` 包含了 `name`，所以用户现在可以只输入一个字符串（名称）。我们将其更改为：

```sdl
function can_enter(person_name: str, place: str) -> str
  using (
    with
      vampire := (SELECT Person FILTER .name = person_name LIMIT 1),
      enter_place := (SELECT HasNameAndCoffins FILTER .name = place LIMIT 1)
    SELECT vampire.name ++ ' can enter.' IF enter_place.coffins > 0 ELSE vampire.name ++ ' cannot enter.'
  );
```

现在我们输入 `can_enter('Count Dracula', 'Munich')` 可以得到 `'Count Dracula cannot enter.'`。这是合理的：德古拉没有带来任何棺材到那里（慕尼黑，Munich）。

最后，我们来创建我们的 `change_coffins` 函数。很容易：

```sdl
function change_coffins(place_name: str, number: int16) -> HasNameAndCoffins
  using (
    UPDATE HasNameAndCoffins FILTER .name = place_name
    SET {
      coffins := .coffins + number
    }
  );
```

现在让我们给 `The Demeter` 船放上一些棺材。

```edgeql
SELECT change_coffins('The Demeter', 10);
```

然后我们来确保一下函数的执行：

```edgeql
SELECT Ship {
  name,
  coffins,
};
```

我们得到：`{default::Ship {name: 'The Demeter', coffins: 10}}`。德米特号（The Demeter）得到了它的棺材。

## 别名：创建子类型（Aliases: creating subtypes when you need them）

我们在本书中大量使用了抽象类型。你会注意到抽象类型本身通常是由非常普遍的概念构成的：`Person`，`HasNameAndCoffins` 等等。在现实生活中的数据库中，你可能会以 `HasEmail`、`HasID` 等形式看到它们，它们被扩展为子类型。别名也可以创建子类型，它们使用 `:=` 而不是 `extending` 且从完整类型中提取。
We've used abstract types a lot in this book. You'll notice that abstract types by themselves are generally made from very general concepts: `Person`, `HasNameAndCoffins`, etc. In databases in real life you'll probably see them in the forms `HasEmail`, `HasID` and so on, which get extended to make subtypes. Aliases also make subtypes, except they use `:=` instead of `extending` and draw from full types.

让我们也为我们的架构（schema）创建一个别名。再来看一下德米特号（The Demeter），船从保加利亚（Bulgaria）的瓦尔纳（Varna）出发，抵达伦敦（London）。让我们想象在我们的游戏中，我们已经将瓦尔纳建成了一个供角色们探索的大港口，并且正在改变架构以反映这一点。现在我们的 `Crewman` 类型看起来像这样：

```sdl
type Crewman extending HasNumber, Person {
}
```

想象一下，出于某种原因，我们想要一个 `CrewmanInBulgaria` 别名，因为保加利亚人互相称对方为“Gospodin”而不是“Mr.”，我们的游戏需要反映这一点。无论何时我们的船员出现在保加利亚时，都将被称为“Gospodin (+名字)”。然我们再加上一个可计算的 `current_location`，并链接到名为保加利亚的 `Place` 类型。如下所示：

```sdl
alias CrewmanInBulgaria := Crewman {
  name := 'Gospodin ' ++ .name,
  current_location := (SELECT Place filter .name = 'Bulgaria'),
}
```

你可能马上注意到了别名（alias）里面的 `name` 和 `current_location` 被逗号隔开了，而不是分号。它表明这不是在创建新类型：它只是在现有的 `Crewman` 类型之上创建了一个 _shape_。出于同样的原因，您不能执行 `INSERT CrewmanInBulgaria`，因为并没有这样的类型。EdgeDB 将给出错误：

```
error: cannot insert into expression alias 'default::CrewmanInBulgaria'
```

所以所有的插入仍然是通过 `Crewman` 类型完成的。但是因为别名（alias）是一个子类型（subtype）和一个形状（shape），所以我们可以像选择其他任何东西一样选择它。现在让我们添加保加利亚：

```edgeql
INSERT Country {
  name := 'Bulgaria'
};
```

然后选择这个别名看看我们得到了什么：

```edgeql
SELECT CrewmanInBulgaria {
  name,
  current_location: {
    name
  }
};
```

现在我们在别名 `CrewmanInBulgaria` 下看到了相同的 `Crewman` 类型：在他们的名字中添加了 _Gospodin_ 并链接到了我们刚刚插入的 `Country` 对象。

```
{
  default::Crewman {
    name: 'Gospodin Crewman 0',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 1',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 2',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 3',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 4',
    current_location: default::Country {name: 'Bulgaria'},
  },
  default::Crewman {
    name: 'Gospodin Crewman 5',
    current_location: default::Country {name: 'Bulgaria'},
  },
}
```

[别名的文档](https://www.edgedb.com/docs/cheatsheet/aliases/) 中提到它们允许你使用“来自 GraphQL 的 EdgeQL（表达式、聚合函数、反向链接导航）的全部功能”，所以如果你经常使用 GraphQL，请记住别名。

## 为查询中的类型创建新名称（本地表达式别名）Creating new names for types in a query (local expression aliases)

有趣的是，当我们编写 `alias CrewmanInBulgaria := Crewman` 时，只是使用了 `:=` 来声明我们的别名。那么是否可以在查询中做类似的事情？答案是肯定的：我们可以使用 `WITH`，然后为现有类型指定一个新名称。（实际上，我们一直在使用的关键字 `WITH` 被定义为“[用于定义别名的块](https://www.edgedb.com/docs/edgeql/statements/with)”）。以这样一个简单的查询为例，它显示了德古拉伯爵和他的奴隶们的名字：

```edgeql
SELECT Vampire {
  name,
  slaves: {
    name
  }
};
```

如果我们想使用 `WITH` 创建一个与 `Vampire` 相同的新类型，我们就这样做：

```edgeql
WITH Drac := Vampire,
SELECT Drac {
  name,
  slaves: {
    name
  }
};
```

到目前为止，没什么特别的，因为输出是相同的：

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

但它变得有用的地方是当使用这种新类型对创建它的类型执行操作或比较时。它与 `DETACHED` 相同，但我们给它起了一个名字，使用起来更加灵活。

所以让我们来试一下。我们假装我们正在测试我们的游戏引擎。现在我们的 `fight()` 函数非常简单，但让我们假设它很复杂并且需要大量测试。下面是 `fight()` 当前的定义：

```sdl
function fight(one: Person, two: Person) -> str
  using (
    SELECT one.name ++ ' wins!' IF one.strength > two.strength ELSE two.name ++ ' wins!'
  );
```

但是出于调试目的，最好有更多信息。让我们创建相同的函数，但将其命名为 `fight_2()` 并添加更多关于谁在与谁战斗的信息。

```edgeql
CREATE FUNCTION fight_2(one: Person, two: Person) -> str
  USING (
    SELECT one.name ++ ' fights ' ++ two.name ++ '. ' ++ one.name ++ ' wins!'
      IF one.strength > two.strength
      ELSE one.name ++ ' fights ' ++ two.name ++ '. ' ++ two.name ++ ' wins!'
  );
```

所以我们让 `MinorVampire` 们互相争斗，看看我们会得到什么结果。我们有四个 `MinorVampire`（三个女吸血鬼加上露西）。首先让我们将 `MinorVampire` 类型放入函数中，看看我们得到了什么。试着想象输出会是什么。

```edgeql
SELECT fight_2(MinorVampire, MinorVampire);
```

所以这个输出是……

……

```
{
  'Lucy Westenra fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

该函数只使用了四次，因为每次进入函数的是只有一个对象的集合……每个 `MinorVampire` 都在和她自己战斗。这可能不是我们想要的。现在让我们用本地类型别名在尝试一下。

```edgeql
WITH M := MinorVampire,
SELECT fight_2(M, MinorVampire);
```

顺便说一句，这实际上与下面的内容完全相同：

```edgeql
SELECT fight_2(MinorVampire, DETACHED MinorVampire);
```

现在，输出很长了：

```
{
  'Lucy Westenra fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 2 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 3 fights Lucy Westenra. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 1. Lucy Westenra wins!',
  'Woman 1 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Lucy Westenra fights Woman 2. Lucy Westenra wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 2 fights Woman 2. Woman 2 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
  'Lucy Westenra fights Woman 3. Lucy Westenra wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Woman 3. Woman 3 wins!',
}
```

我们成功地让每个 `MinorVampire` 都与其他 `MinorVampire` 进行了战斗，但仍然有 `MinorVampire` 和自己战斗（露西对露西，女人 1 对女人 1，等等）。这就是本地类型别名的便利之处：例如，我们可以对其进行过滤。现在我们将过滤写为仅对彼此不同的对象使用 `fight_2()`：

```edgeql
WITH M := MinorVampire,
SELECT fight_2(M, MinorVampire) FILTER M != MinorVampire;
```

现在我们终于让每个 `MinorVampire` 都与其他 `MinorVampire` 进行了战斗，且没有重复。

```
{
  'Lucy Westenra fights Woman 1. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 2. Lucy Westenra wins!',
  'Lucy Westenra fights Woman 3. Lucy Westenra wins!',
  'Woman 1 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 1 fights Woman 2. Woman 2 wins!',
  'Woman 1 fights Woman 3. Woman 3 wins!',
  'Woman 2 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 2 fights Woman 1. Woman 1 wins!',
  'Woman 2 fights Woman 3. Woman 3 wins!',
  'Woman 3 fights Lucy Westenra. Lucy Westenra wins!',
  'Woman 3 fights Woman 1. Woman 1 wins!',
  'Woman 3 fights Woman 2. Woman 2 wins!',
}
```

完美！

仅使用 `DETACHED` 是行不通的：`SELECT Fight_2(MinorVampire, DETACHED MinorVampire) FILTER MinorVampire != DETACHED MinorVampire;` 是做不到的，因为第一个 `DETACHED MinorVampire` 不是变量名。如果没有该名称可供访问，下一个 `DETACHED MinorVampire` 只会是一个新的 `DETACHED MinorVampire`，与另一个没有关系。

那么如何用与我们对 `CrewmanInBulgaria` 别名所做的相同的方式来添加链接和属性呢？我们也可以通过使用 `SELECT` 然后在 `{}` 中添加任何你想要的新链接和属性来做到这一点。下面是一个简单的例子：

```edgeql
WITH NPCExtraInfo := (
    SELECT NPC {
      would_win_against_dracula := .strength > Vampire.strength
    }
  )
SELECT NPCExtraInfo {
  name,
  would_win_against_dracula
};
```

结果如下。看起来没有人能赢德古拉：

```
{
  default::NPC {name: 'Jonathan Harker', would_win_against_dracula: {false}},
  default::NPC {name: 'Renfield', would_win_against_dracula: {false}},
  default::NPC {name: 'The innkeeper', would_win_against_dracula: {false}},
  default::NPC {name: 'Mina Murray', would_win_against_dracula: {false}},
  default::NPC {name: 'John Seward', would_win_against_dracula: {false}},
  default::NPC {name: 'Quincey Morris', would_win_against_dracula: {false}},
  default::NPC {name: 'Arthur Holmwood', would_win_against_dracula: {false}},
  default::NPC {name: 'Abraham Van Helsing', would_win_against_dracula: {false}},
  default::NPC {name: 'Lucy Westenra', would_win_against_dracula: {false}},
}
```

现在假设德古拉已经实现了他的所有目标，现在统治了伦敦，让我们为其创建一个快速类型别名。我们将创建一个叫做 `DraculaKingOfLondon` 的快速新类型，包含一个指向 `subjects`（= 国王统治的人）的链接，该链接将是每个去过伦敦的 `Person`。然后我们将选择这种类型，并计算有多少 `subjects`。如下所示：

```edgeql
WITH DraculaKingOfLondon := (
    SELECT Vampire {
      name := .name ++ ', King of London',
      subjects := (SELECT Person FILTER 'London' in .places_visited.name),
    }
  )
SELECT DraculaKingOfLondon {
  name,
  subjects: {name},
  number_of_subjects := count(.subjects)
};
```

这是输出：

```
{
  default::Vampire {
    name: 'Count Dracula, King of London',
    subjects: {
      default::NPC {name: 'Jonathan Harker'},
      default::NPC {name: 'Renfield'},
      default::NPC {name: 'The innkeeper'},
      default::NPC {name: 'Mina Murray'},
      default::NPC {name: 'John Seward'},
      default::NPC {name: 'Quincey Morris'},
      default::NPC {name: 'Arthur Holmwood'},
      default::NPC {name: 'Abraham Van Helsing'},
      default::NPC {name: 'Lucy Westenra'},
    },
    number_of_subjects: 9,
  },
}
```

[点击这里查看第 17 章相关代码](code.md)

<!-- quiz-start -->

##小测验

1. 如何显示每个 NPC 的名称、力量值、所访问城市的名称和人口，以及年龄（如果年龄 = `{}`，则显示 0）？试试只用单行代码。

2. 上题的查询显示了很多没有任何上下文的数字。我们应该做些什么使其更友好？

3. 伦菲尔德（Renfield）现在已经死了，需要一个 `last_appearance`。尝试编写一个名为 `make_dead(person_name: str, date: str) -> Person` 的函数，你只需要输入角色的名字和日期就即可。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森侦探。_
