---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# 第一章 - 乔纳森·哈克前往特兰西瓦尼亚

在小说《德古拉》的开头，我们看到主人公乔纳森·哈克（Jonathan Harker）是一位年轻的律师，他正要去见一位客户。客户是一位生活在东欧某个地方、名叫德古拉伯爵（Count Dracula）的富人。乔纳森并不知道德古拉是一只吸血鬼，所以他还很享受前往欧洲的旅行。小说开始于乔纳森旅行时写下的日记。内容如下，其中粗体的部分适合存入数据库：


> **5月3日** **比斯特里察（Bistritz）**——**5月1日** **8:35 P.M.** 离开**慕尼黑（Munich）**，次日清晨抵达**维也纳（Vienna）**；本应在 6:46 抵达，但火车晚点了一个小时。从我在火车上瞥到的来看，**布达佩斯（Buda-Pesth）** 似乎是一个很棒的地方。

## 架构，对象类型（Schema, object types）

目前我们已经有很多信息了，它们帮助我们开始思考我们的数据库架构。EdgeDB 使用的语言称为 EdgeQL，它用于定义（define）、变换（mutate）和查询（query）数据。它的内部是 [SDL (schema definition language)](https://edgedb.com/docs/edgeql/sdl/index#ref-eql-sdl)，它使迁移变得更加容易，我们将在本书中学习。到目前为止，我们的架构需要以下内容：

- 城市或位置类型。我们可以创建的这些类型称为 [对象类型](https://www.edgedb.com/docs/datamodel/objects#object-types)，由属性和链接组成。一个城市类型应该有什么样的属性？可能是名字和地理位置，以及有时会使用的不同的名称或拼写。比如，Bistritz（比斯特里察）现在叫做 Bistrița（该城市在罗马尼亚），而 Buda-Pesth（布达佩斯）现在会被写为 Budapest。
- 人类类型。我们需要赋予它姓名，以及一种方式去追踪他所访问过的地方。

创建一个类型，只需要使用关键字 `type` 并跟随对应的类型名及花括号 `{}`。类型 `Person`（人）的创建如下所示：

```sdl
type Person {
}
```

现在你已经完成了一个类型的创建，但里面还什么都没有。我们可以在花括号中为 `Person` 类型添加属性。如果属性是必须的，我们使用 `required property` 前置在属性名前；如果属性是可选的，我们使用  `property`。

```sdl
type Person {
  required property name -> str;
  property places_visited -> array<str>;
}
```

属性 `required property name` 意味着：使用 `Person` 这个类型创建的对象必须保证拥有一个名称，即你不能创建一个没有名称/姓名的 `Person` 的对象，否则你会看到这样的错误提示：

```
MissingRequiredError: missing value for required property default::Person.name
```

`str` 是指一个字符串，可以包含在单引号内：`'Jonathan Harker'` 或双引号内：`"Jonathan Harker"`。引号前使用转义字符 `\` 可使 EdgeDB 将引号视为一个字母打印出来：`'Jonathan Harker\'s journal'`。

`array` 是相同类型的集合，即数组，我们这里的数组是一个 `str` 的数组。我们希望它看起来像这样：`["Bistritz", "Vienna", "Buda-Pesth"]`。这样便于我们稍后搜索并查看哪个角色访问过哪里。

`places_visited` 不是一个 `required` 属性，因为我们之后可能会添加不会去任何地方的配角。也许会是“比斯特里察的某个客栈老板”或是其他什么人，我们并不了解或关心他的 `places_visited`。

现在让我们继续创建城市类型：

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
}
```

这是类似的，只是带有字符串的属性。《德古拉》这本书出版于 1897 年，当时有些城市的名称拼写与现在有所不同。书中所有的城市都有名称（这就是为什么我们要在属性 `name` 前用 `required`），但有些城市并不需要不同的现代名称。比如，维也纳（Vienna）至今仍然拼写为维也纳（Vienna）。我们希望我们的游戏会将书中的城市名称与其现代名称关联起来，以便现在的我们可以轻松地在地图上找到它们。

## 迁移（Migration）

目前我们还未创建我们的数据库。在 [安装 EdgeDB](https://edgedb.com/download) 后，还有两个小步骤需要做。首先，我们需要键入关键字 `CREATE DATABASE` 及数据库的名称（比如：dracula）来创建一个数据库：

```edgeql
CREATE DATABASE dracula;
```

然后我们键入 `\c dracula` 去连接该数据库。

最后，我们需要做一个迁移。这将为数据库提供我们可以开始与之交互所需的结构。 EdgeDB 的迁移并不困难：

- 首先，键入 `START MIGRATION TO {}`。
- 在这个里面你至少要添加一个 `module`，你的类型才可以被访问。一个模块是一个命名空间，是相似类型聚集在一起的地方。`::` 左边的部分是模块的名称，右边是模块里包含的类型。如果你写了 `module default`，然后写 `type Person`，`Person` 类型将是 `default::Person`。因此，例如，当你看到像 `std::bytes` 这样的类型时，它是指 `std`（标准库）中的类型 `bytes`。
- 然后添加我们上面提及的类型，并以`}`结尾来结束该表达块。然后在此之外，键入 `POPULATE MIGRATION` 以添加数据。
- 最后，键入 `COMMIT MIGRATION` 以完成迁移。

当然，除此之外自然还有很多其他命令，尽管我们在本书中并不需要它们，但你可以将下面的四个页面添加至书签以供今后需要时使用：

- [Admin commands](https://www.edgedb.com/docs/cheatsheet/admin)：创建用户角色，设置密码，配置端口等。
- [CLI commands](https://www.edgedb.com/docs/cheatsheet/cli)：创建数据库，角色，为角色设置密码，连接数据库等。
- [REPL commands](https://www.edgedb.com/docs/cheatsheet/repl)：主要介绍我们将在本书中使用的许多命令的快捷方式。
- [Various commands](https://www.edgedb.com/docs/edgeql/statements/tx_rollback#rollback)：关于回滚事务、保存点声明等等。

我们还提供了以下编辑器的语法高亮插件，如果你愿意，可以通过相应的链接进行下载：[Atom](https://atom.io/packages/edgedb)，[Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb)，[Sublime Text](https://packagecontrol.io/packages/EdgeDB)，[Vim](https://github.com/edgedb/edgedb-vim).

这是我们刚刚创建的类型 `City`：

```edgeql
type City {
  required property name -> str;
  property modern_name -> str;
}
```

## 选择（Selecting）

在 EdgeDB 里，有三个包含了 `=` 的操作符，它们分别是：

- `:=` 用于声明；
- `=` 用于检查相等性 (注意：不是 `==`)；
- `!=` 是 `=` 相反的意思。

让我们用 `SELECT` 来试试这些操作符。`SELECT` 是 EdgeDB 里主要的查询命令，你可以使用它基于其后面的输入语句查询相应的结果。

顺便说一句，EdgeDB 里的关键字不区分大小写，因此使用 `SELECT`，`select` 和 `SeLeCT` 都是一样的效果。但是使用大写字母是数据库的常规做法，因此我们将继续以这种方式使用它们。

首先，我们选择一个字符串：

```edgeql
SELECT 'Jonathan Harker\'s journey begins.';
```

毫无意外，它将返回 `{'Jonathan Harker\'s journey begins.'}`，但你是否注意到它是在一个花括号 `{}` 里返回的？这个 `{}` 意味着它是一个集合，并且事实上，[在 EdgeDB 里一切都是一个集合](https://www.edgedb.com/docs/edgeql/overview#everything-is-a-set)（请确保记住这一点）。这也是为什么 EdgedB 里没有 null：在其他语言里会有 null，但在 EdgeDB 里你只会得到一个空集：`{}`。

接下来，我们来用 `:=` 为一个变量赋值：

```edgeql
SELECT jonathans_name := 'Jonathan Harker';
```

它只是返回了我们给它的内容 `{'Jonathan Harker'}`。但是这次返回的是我们赋予的变量名为 `jonathans_name` 的字符串。

现在让我们用这个变量做一些事情。我们可以通过用关键字 `WITH` 来使用这个变量，然后用它和 `'Count Dracula'` 进行比较：

```edgeql
WITH jonathans_name := 'Jonathan Harker',
SELECT jonathans_name != 'Count Dracula';
```

输出结果是 `{true}`。当然，你也可以写 `SELECT 'Jonathan Harker' != 'Count Dracula'` 而得到同样的结果。接下来我们马上就会对我们用 `:=` 赋值的变量做一些事情。

## 插入对象（Inserting objects）

回到架构设计上，稍后我们会考虑给我们游戏中的城市增加时区和地理位置等信息。但现在，我们先来使用 `INSERT` 向数据库中插入一些数据。

别忘了要用逗号分割开各个属性，并用分号结束 `INSERT` 语句。此外，EdgeDB 同样倾向用两个空格作为缩进。

```edgeql
INSERT City {
  name := 'Munich',
};

INSERT City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};
```

注意：花括号里最后一项末尾的逗号是可选的——你可以加上逗号，也可以不加逗号。本书里，我们有时会在末尾添加一个逗号，有时也会将其省略。

最后，`Person` 的插入如下所示：

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Vienna", "Buda-Pesth"],
};
```

但是稍等，这个插入将不会链接到任何我们已经插入到数据库里的 `City`。这是我们的架构需要改进的地方：

- 我们有一个 `Person` 类型和一个 `City` 类型，
- `Person` 类型含有属性 `places_visited`，用来展示访问过的城市的名称，但它们只是数组中的字符串。我们最好能以某种方式将这个属性链接到 `City` 类型上。

所以这里我们先不急于做 `Person` 的插入。我们先来调整一下 `Person` 类型的定义，将 `array<str>` 从 `property` 更改为指向 `City` 类型的 `multi link`。这将使 `Person` 和 `City` 连接起来。

但首先让我们先仔细看看当我们使用 `INSERT` 时到底发生了什么。

正如你所见，`str` 适用于像 ț 这样的 unicode 字母。甚至表情符号和特殊字符也都 OK：如果你愿意，你甚至可以创建一个名为“🤠”或“(╯°□°)╯︵ ┻━┻”的 `City`。

EdgeDB 也有一个字节文字类型，它为你提供字符串的字节。这主要用于人们在保存到文件时不需要查看的原始数据。它们必须是 1 个字节长的字符。

你可以通过在字符串前添加一个 `b` 来创建一个字节文字：

```edgeql-repl
edgedb> SELECT b'Bistritz';
{b'Bistritz'}
```

因为字符必须是 1 个字节，因此只有 ASCII 才适用于这种类型。所以如果 `modern_name` 是字节类型且名字中有类似 `ț` 这样的字符，将会产生错误：

```edgeql-repl
edgedb> SELECT b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
```

每当你 `INSERT` 一个项目（an item，一条数据）时，EdgeDB 都会给你一个 `uuid`。这是每个项目的唯一编号，类似于：

```
{Object {id: d2af670c-f1d6-11ea-a30f-8b40bc5413e0}}
```

当你使用 `SELECT` 选择一个类型时也会显示类似的内容。只需键入 `SELECT` 和相应类型就会显示该类型下所有的 `uuid`。让我们看看到目前为止创建的所有城市：

```edgeql
SELECT City;
```

这个命令返回了我们三个项目：

```
{
  Object {id: d2b64e00-f1d6-11ea-a30f-1f161d0b15ae},
  Object {id: d2c023b2-f1d6-11ea-a30f-e3069a47b57e},
  Object {id: d37bc838-f1d6-11ea-a30f-afb031317264},
}
```

这仅说明我们已经有三个 `City` 类型的对象。如果想查看对象内部，我们可以在查询语句里添加需要查看的属性或链接的名称。这称为“描述我们想要的数据的 [形状](https://www.edgedb.com/docs/edgeql/expressions/shapes/#ref-eql-expr-shapes)”。我们选择所有 `City` 并通过下面这个查询显示他们的  `modern_name`：

```edgeql
SELECT City {
  modern_name,
};
```

再一次说明，你不是必须在 `modern_name` 后面使用逗号，因为它是查询的末尾。

相信你还记得，书中提到的其中一个城市（维也纳 Vienna）并没有与以前不同的现代名称 `modern_name`。但是它仍会显示一个“空集”，因为在 EdgeDB 中，所有数值都是元素的集合，即使里面没有任何东西。这里是上面查询的结果：

```
{Object {modern_name: {}}, Object {modern_name: 'Budapest'}, Object {modern_name: 'Bistrița'}}
```

因此有对象的 `modern_name` 是空集，同时另外两个对象的 `modern_name` 都有各自的名字。这再一次向我们展示了 EdgeDB 不存在其他一些语言里的 `null`，如果什么都没有，它会返回一个空的集合。 

现在第一个对象看起来是个谜（因为它不存在 `modern_name`），因此我们将在查询语句中添加 `name`，以便我们可以确认它就是城市“维也纳”（Vienna）：

```edgeql
SELECT City {
  name,
  modern_name
};
```

由此得到输出：

```
{
  Object {name: 'Vienna', modern_name: {}},
  Object {name: 'Bistritz', modern_name: 'Bistrița'},
  Object {name: 'Buda-Pesth', modern_name: 'Budapest'}
}
```

如果你只想返回类型的单个部分，而不是整个对象结构，你可以在类型名称后使用 `.` 。比如，`SELECT City.modern_name` 将输出结果：

```
{'Budapest', 'Bistrița'}
```

这种表达式类型称为 _路径表达式_ 或 _路径（path)_，因为它是通往内部数值的直接路径。并且每个 `.` 都意味着前往再下一个路径，如果还有再深一层数值需要跟踪的话。

如果需要，你也可以在所需名称后使用 `:=`，将诸如 `modern_name` 之类的属性名称更改为成任何其他所需名称。这些名称将作为变量名在输出中显示出来。例如：

```edgeql
SELECT City {
  name_in_dracula := .name,
  name_today := .modern_name,
};
```

输出结果为：

```
{
  Object {name_in_dracula: 'Munich', name_today: {}},
  Object {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest'},
  Object {name_in_dracula: 'Bistritz', name_today: 'Bistrița'},
}
```

架构里的内容不会因此发生变化——它只是一个在查询中使用的快速变量名。

顺便说一下，`.name` 是 `City.name` 的缩写。你也可以每次都写 `City.name`（这被称为 _fully qualified name_），但不是必需的。

所以，如果你可以对 `.name` 做一个快速索引的属性 `name_in_dracula`，我们也可以做其他类似的事情吗？我们确实可以，为了简单易懂，我们在这里举一个例子：

```edgeql
SELECT City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'This is a city in the book Dracula'
};
```

输出结果是：

```
{
  Object {name_in_dracula: 'Munich', name_today: {}, oh_and_by_the_way: 'This is a city in the book Dracula'},
  Object {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest', oh_and_by_the_way: 'This is a city in the book Dracula'},
  Object {name_in_dracula: 'Bistritz', name_today: 'Bistrița', oh_and_by_the_way: 'This is a city in the book Dracula'},
}
```

注意：`oh_and_by_the_way` 的类型是 `str`，即使我们不是必须说明它。EdgeDB 是强类型的：一切都需要一个类型，它不会尝试将不同类型混在一起。因此，如果你写 `SELECT 'Jonathan Harker' + 8;`，EdgeDB 会直接拒绝执行并给出错误提示 `QueryError: operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'`。

另一方面，EdgeDB 可以使用“类型推断”来猜测类型，这就是 EdgeDB 在上面的例子里所做的：它知道我们正在创建一个 `str`。之后我们很快就会看到如何改变类型并使用不同的类型。

## 链接（Links）

所以现在剩下要做的最后一件事是将 `Person` 中名为 `places_visited` 的 `property` 更改为 `link`。现在，`places_visited` 为我们提供了我们想要的城市名称，但将 `Person` 和 `City` 链接在一起才更有意义。毕竟，`City` 类型里面有 `.name`，链接它们比重写 `Person` 中的所有内容更好些。我们将把 `Person` 改成这样：

```sdl
type Person {
  required property name -> str;
  multi link places_visited -> City;
}
```

我们在 `link` 前写了 `multi`，因为一个 `Person` 应该可以链接不止一个 `City`。`multi` 的相反是 `single`，使用 `single` 意味着仅允许链接一个对象。因为 `single` 是默认的，所以如果你仅仅写 `link`，EdgeDB 将把它视为 `single`。

现在当我们插入乔纳森·哈克（Jonathan Harker）时，他将被链接到类型 `City`。别忘了属性 `places_visited` 不是 `required` 的，所以我们仍然可以在插入时仅使用他的名字进行创建：

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
};
```

这会生成一个 `Person` 类型的对象，其 `places_visited` 属性被链接到 `City` 类型且没有任何内容。让我们看看里面是什么：

```edgeql
SELECT Person {
  name,
  places_visited
};
```

执行后会输出：`{Object {name: 'Jonathan Harker', places_visited: {}}}`

但是我们想要把乔纳森（Jonathan）关联到他所去过的城市。我们在做 `INSERT` 的时候，可以将 `places_visited` 变为 `places_visited := City`：

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

我们没有做任何过滤，所以 EdgeDB 会把所有 `City` 类型的对象都放进来。现在让我们看看乔纳森（Jonathan）都去了哪些地方。尝试一下下面的代码，你会发现这还不完全是我们需要的：

```edgeql
SELECT Person {
  name,
  places_visited
};
```

执行后的输出:

```
  Object {
    name: 'Jonathan Harker',
    places_visited: {
      Object {id: 5ba2c9b2-f64d-11ea-acc7-d3a96742e345},
      Object {id: 5bbb2368-f64d-11ea-acc7-ffb002eabe1a},
      Object {id: 5bc2bba0-f64d-11ea-acc7-8b468bbfae39},
    },
  },
```

和我们理想的结果已经很接近了！因为我们没有提到 `City` 里的任何属性，所以我们只能得到一堆 Object ID。现在我们仅仅需要让 EdgeDB 知道我们实际上是想看到 `City` 类型中的 `name` 属性。为了得到想要的结果，我们需要在 `places_visited` 后面添加一个冒号，并将 `name` 放到随后的花括号中：

```edgeql
select Person {
  name,
  places_visited: {
    name
  }
};
```

成功了！现在我们可以从输出结果中得到我们想要的了：

```
  Object {
    name: 'Jonathan Harker',
    places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}},
  },
```

当然，乔纳森·哈克（Jonathan Harker）已经成功被插入到数据库中并关联了每一个造访过的城市。现在我们只有三个 `City` 对象，所以这还没有什么问题。但是稍后我们将有更多的城市，并且不能对其他所有角色都使用 `places_visited := City`（因为他们造访过的城市列表并不一样）。为此，我们将需要用到 `FILTER`，我们将在下一章中学习如何使用它。

[→ 点击这里查看到第 1 章为止的所有代码](code.md)

<!-- quiz-start -->

## 小测验

1. 输入下面的代码将会被返回错误，请尝试通过添加一个字符，使其返回结果为 `{true}`。

   ```edgeql
   WITH my_name = 'Timothy',
   SELECT my_name != 'Benjamin';
   ```

2. 请尝试插入一个名为 Constantinople 的 `City`，其现代的命名为 İstanbul。
3. 请尝试显示数据库中所有城市的名称。（提示：使用单行代码就可以做到，并不需要使用 `{}`）
4. 请尝试 SELECT 出所有城市的 `name` 和 `modern_name`，并将 `.name` 显示为 `old_name`，将 `.modern_name` 显示为 `name_now`。
5. 键入 `SelecT City;` 会发生错误吗？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _乔纳森·哈克抵达罗马尼亚_
