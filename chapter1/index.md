---
tags: Object Types, Select, Insert
leadImage: illustration_01.jpg
---

# 第一章 - 乔森纳·哈克前往特兰西瓦尼亚

在本书（《德拉库拉》）的开头，我们看到主人公乔森纳·哈克（Jonathan Harker）是一位年轻的律师，他正要去见一位客户。客户是一位生活在东欧某个地方的富人，名叫德拉库拉伯爵（Count Dracula）。乔森纳尚不知德拉库拉是一个吸血鬼，因此他还沉浸在前往欧洲新地方的旅行中。本书开始于乔森纳旅行时写下的日志。如下，其中粗体的部分适合存入数据库：


> **五月三日**，**比斯特里察（Bistritz）** —— 于 **五月一日** **8:35 P.M.** 离开 **慕尼黑（Munich）**，次日清晨抵达 **维也纳（Vienna）**；本应在 6:46 到达，但火车晚点了一个小时。从我在火车上瞥到的来看，**布达佩斯（Buda-Pesth）** 似乎是一个很棒的地方。

## 架构，对象类型（Schema, object types）

这已经包含了很多信息了，它帮助我们开始思考我们的数据库架构。EdgeDB 使用的语言称为 EdgeQL，它用于定义、变异和查询数据。它的内核是 [SDL（模式定义语言）](https://edgedb.com/docs/edgeql/sdl/index#ref-eql-sdl)，它使迁移变得更加容易，我们将在本书中学习。到目前为止，我们的架构需要以下内容：

- 城市或位置类型。我们可以创建的这些类型称为 [对象类型](https://www.edgedb.com/docs/datamodel/objects#object-types)，由属性和链接组成。一个城市类型应该有什么样的属性？可能是名字和地理位置，以及有时会使用的不同的名称或拼写。比如，Bistritz（比斯特里察）现在叫做 Bistrița（这个城市在罗马尼亚），而 Buda-Pesth（布达佩斯）现在会被写为 Budapest.
- 人类类型。我们需要赋予它姓名，以及一种方式去追踪他所访问过的地方。

在模型里创建一个类型，仅需要使用关键字 `type` 并跟随对应的类型名及花括号 `{}`。人类的类型 `Person` 可如下创建

```sdl
type Person {
}
```

现在你已经完成了一个类型的创建，但里面还什么都没有。我们可以在花括号中为 `Person` 添加属性。如果属性是必须的，我们使用 `required property` 前置在属性名前，如果属性是可选的，我们使用  `property`。

```sdl
type Person {
  required property name -> str;
  property places_visited -> array<str>;
}
```

属性  `required property name` 意味着，使用 `Person` 这个类型创建的对象必须拥有一个名称，即你不能创建一个没有名称的 `Person` 的对象，否则你会看到这样的错误提示：

```
MissingRequiredError: missing value for required property default::Person.name
```

 `str` 是指一个字符串，可以包含在单引号内：`'Jonathan Harker'` 或双引号：`"Jonathan Harker"`。引号前的 `\` 转义字符可使 EdgeDB 将其视为一个字母：`'Jonathan Harker\'s journal'`。

`array` 是相同类型的集合，即数组，我们这里的数组是一个 `str` 的数组。我们希望它看起来像这样：`["Bistritz", "Vienna", "Buda-Pesth"]`。这样便于我们稍后搜索并查看哪个角色访问过哪里。

`places_visited` 不是一个 `required` 属性，因为我们之后可能会添加不会去任何地方的配角。也许会是“比斯特里察的某个客栈老板”或是其他什么人，我们并不了解或关心他的 `places_visited`。

现在让我们继续创建城市类型：

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
}
```

这是类似的，只是带有字符串的属性。《德拉库拉》这本书出版于 1897 年，当时有些城市的名称拼写与现在有所不同。书中所有的城市都有名称（这就是为什么我们要在名称属性前用 `required`），但有些城市并不需要不同的现代名称。比如，维也纳（Vienna）至今仍然拼写为维也纳（Vienna）。我们设想我们的游戏会将书中的城市名称与其现代名称关联起来，以便我们可以轻松地在地图上找到他们。

## 迁移（Migration）

我们尚未创建我们的数据库。 [在安装 EdgeDB 后](https://edgedb.com/download) 我们需要做两个步骤。第一步，我们需要用关键字 `CREATE DATABASE` 及我们要赋予其的名称来创建一个数据库：

```edgeql
CREATE DATABASE dracula;
```

然后我们键入 `\c dracula` 去链接它。

最后，我们需要做一个迁移。这将为数据库提供我们开始与之交互所需的结构。 EdgeDB 的迁移并不困难：

- 首先，键入 `START MIGRATION TO {}`
- 在这个里面你至少要添加一个 `module`，你的类型才可以被访问。一个模块是一个命名空间，是相似类型聚集在一起的地方。`::` 左边的部分是模块的名称，右边是模块里包含的类型。如果你写了 `module default`，然后写 `type Person`，这意味着类型 `Person` 就会是 `default::Person`。因此，例如，当你看到像 `std::bytes` 这样的类型时，这意味着 `std`（标准库）中的类型 `bytes`。
- 然后添加我们上面提及的类型，并以`}`结尾结束该块。然后在此之外，键入 `POPULATE MIGRATION` 以添加数据。
- 最后，键入 `COMMIT MIGRATION` 以完成迁移。

当然，除此之外还有很多其他命令，尽管我们在本书中不需要它们。但你可以将下面的四个页面添加至书签以供今后使用：

- [Admin commands](https://www.edgedb.com/docs/cheatsheet/admin): 创建用户角色，设置密码，配置端口等。
- [CLI commands](https://www.edgedb.com/docs/cheatsheet/cli): 创建数据库，角色，为角色设置密码，链接数据库等。
- [REPL commands](https://www.edgedb.com/docs/cheatsheet/repl): 主要介绍我们将在本书中使用的许多命令的快捷方式。
- [Various commands](https://www.edgedb.com/docs/edgeql/statements/tx_rollback#rollback) 关于回滚事务、保存点声明等。

我们还提供了以下编辑器的语法高亮插件，如果你愿意，可以通过相应的链接进行下载：[Atom](https://atom.io/packages/edgedb)，[Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=magicstack.edgedb)，[Sublime Text](https://packagecontrol.io/packages/EdgeDB)，[Vim](https://github.com/edgedb/edgedb-vim).

这里，我们创建类型 `City` 如下：

```edgeql
type City {
  required property name -> str;
  property modern_name -> str;
}
```

## 选择（Selecting）

在 EdgeDB 里，有三个包含了 `=` 的操作符：

- `:=` 用于声明，
- `=` 用于检查相等性 (不是 `==`),
- `!=` 是 `=` 相反的意思。

让我们用 `SELECT` 来试试这些操作符。`SELECT` 是 EdgeDB 里主要的查询命令，你可以使用它根据其后面的输入语句查询相应的结果。

顺便说一句，EdgeDB 里的关键字不区分大小写，因此使用 `SELECT`，`select` 和 `SeLeCT` 都是一样的效果。但是使用大写字母是数据库的常规做法，因此我们将继续以这种方式使用它们。

首先，我们选择一个字符串：

```edgeql
SELECT 'Jonathan Harker\'s journey begins.';
```

这将返回 `{'Jonathan Harker\'s journey begins.'}`，这并不惊讶。你是否注意到它是在一个花括号 `{}` 里返回的？这个 `{}` 意味着它是一个集合，并且事实上，[在 EdgeDB 里一切都是一个集合](https://www.edgedb.com/docs/edgeql/overview#everything-is-a-set)（请确保记住这一点）。这也是为什么 EdgedB 里没有 null：在其他语言里会有 null，在 EdgeDB 里你只会得到一个空集：`{}`。

接下来，我们用 `:=` 为一个变量赋值：

```edgeql
SELECT jonathans_name := 'Jonathan Harker';
```

这只是返回了我们给它的内容 `{'Jonathan Harker'}`。但是这次返回的是我们分配的名为 `jonathans_name` 的字符串。

现在让我们用这个变量做一些事情。我们可以通过关键字 `WITH` 使用这个变量，然后将它和 `'Count Dracula'` 进行比较：

```edgeql
WITH jonathans_name := 'Jonathan Harker',
SELECT jonathans_name != 'Count Dracula';
```

输出结果是 `{true}`。当然，你可以只是写 `SELECT 'Jonathan Harker' != 'Count Dracula'` 而得到同样的额结果。很快我们就会对我们用 `:=` 分配的变量做一些事情。

## 插入对象（Inserting objects）

让我们回到架构上。稍后我们可以考虑为我们想象的游戏添加城市的时区和位置。但与此同时，我们将使用 `INSERT` 向数据库添加一些项目。

别忘了用逗号分割各个属性，并用分号结束 `INSERT`。EdgeDB 也倾向用两个空格作为缩进。

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

注意最后一项末尾的逗号是可选的 —— 你可以加上逗号，也可以不加逗号。在这里，我们有时会在末尾添加一个逗号，而在其他时候则将其省略。

最后，`Person` 的插入看起来像这样：

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := ["Bistritz", "Vienna", "Buda-Pesth"],
};
```

但是请稍等，这个插入将不会链接到任何我们已经做过插入的 `City`。这是我们的架构需要改进的地方：

- 我们拥有一个 `Person` 类型和一个 `City` 类型,
- `Person` 类型具有带有城市名称的 `places_visited` 属性，但它们只是数组中的字符串。最好是能以某种方式将这个属性链接到 `City` 类型。

所以我们先不要做 `Person` 的插入。我们将先通过将 `property` 的 `array<str>` 更改为 `City` 类型的、称为 `multi link` 的内容来修复 `Person` 类型。这将使他们连接在一起。

但首先让我们仔细看看当我们使用 `INSERT` 时发生了什么。

正如你所见，`str` 适用于像 ț 这样的 unicode 字母。甚至表情符号和特殊字符也很 OK：如果你愿意，你甚至可以创建一个名为“🤠”或“(╯°□°)╯︵ ┻━┻”的 `City`。

EdgeDB 也有一个字节文字类型，它为你提供字符串的字节。这主要用于人们在保存到文件时不需要查看的原始数据。它们必须是 1 个字节长的字符。

你可以通过在字符串前添加一个 `b` 来创建一个字节文字：

```edgeql-repl
edgedb> SELECT b'Bistritz';
{b'Bistritz'}
```

因为字符必须是 1 个字节，只有 ASCII 才适用于这种类型。所以使用字节类型的 `modern_name` 中的名称如果有类似 `ț` 这样的字符，将会产生错误：

```edgeql-repl
edgedb> SELECT b'Bistrița';
error: invalid bytes literal: character 'ț' is unexpected, only ascii chars are allowed in bytes literals
```

每当你 `INSERT` 一个项目时，EdgeDB 都会给你一个 `uuid`。这是每个项目的唯一编号，类似于：

```
{Object {id: d2af670c-f1d6-11ea-a30f-8b40bc5413e0}}
```

这也是你使用 `SELECT` 选择一个类型时会显示的内容。只需键入带有类型的 `SELECT` 就会显示该类型的所有 `uuid`。让我们看看到目前为止我们拥有的所有城市：

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

这仅仅是告诉我有三个 `City` 类型的对象。
This only tells us that there are three objects of type `City`. To see inside them, we can add property or link names to the query. This is called describing the [shape](https://www.edgedb.com/docs/edgeql/expressions/shapes/#ref-eql-expr-shapes) of the data we want. We'll select all `City` types and display their `modern_name` with this query:

```edgeql
SELECT City {
  modern_name,
};
```

Once again, you don't need the comma after `modern_name` because it's at the end of the query.

You will remember that one of our cities (Vienna) doesn't have anything for `modern_name`. But it still shows up as an "empty set", because every value in EdgeDB is a set of elements, even if there's nothing inside. Here is the result:

```
{Object {modern_name: {}}, Object {modern_name: 'Budapest'}, Object {modern_name: 'Bistrița'}}
```

So there is some object with an empty set for `modern_name`, while the other two have a name. This shows us that EdgeDB doesn't have `null` like in some languages: if nothing is there, it will return an empty set.

The first object is a mystery so we'll add `name` to the query so we can see that it's the city of Vienna:

```edgeql
SELECT City {
  name,
  modern_name
};
```

This gives the output:

```
{
  Object {name: 'Vienna', modern_name: {}},
  Object {name: 'Bistritz', modern_name: 'Bistrița'},
  Object {name: 'Buda-Pesth', modern_name: 'Budapest'}
}
```

If you just want to return a single part of a type without the object structure, you can use `.` after the type name. For example, `SELECT City.modern_name` will give this output:

```
{'Budapest', 'Bistrița'}
```

This type of expression is called a _path expression_ or a _path_, because it is the direct path to the values inside. And each `.` moves on to the next path, if there is another one to follow.

You can also change property names like `modern_name` to any other name if you want by using `:=` after the name you want. Those names you choose become the variable names that are displayed. For example:

```edgeql
SELECT City {
  name_in_dracula := .name,
  name_today := .modern_name,
};
```

This prints:

```
{
  Object {name_in_dracula: 'Munich', name_today: {}},
  Object {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest'},
  Object {name_in_dracula: 'Bistritz', name_today: 'Bistrița'},
}
```

This will not change anything inside the schema - it's just a quick variable name to use in a query.

By the way, `.name` is short for `City.name`. You can also write `City.name` each time (that's called the _fully qualified name_), but it's not required.

So if you can make a quick `name_in_dracula` property from `.name`, can we make other things too? Indeed we can. For the moment we'll just keep it simple but here is one example:

```edgeql
SELECT City {
  name_in_dracula := .name,
  name_today := .modern_name,
  oh_and_by_the_way := 'This is a city in the book Dracula'
};
```

And here is the output:

```
{
  Object {name_in_dracula: 'Munich', name_today: {}, oh_and_by_the_way: 'This is a city in the book Dracula'},
  Object {name_in_dracula: 'Buda-Pesth', name_today: 'Budapest', oh_and_by_the_way: 'This is a city in the book Dracula'},
  Object {name_in_dracula: 'Bistritz', name_today: 'Bistrița', oh_and_by_the_way: 'This is a city in the book Dracula'},
}
```

Also note that `oh_and_by_the_way` is of type `str` even though we didn't have to tell it. EdgeDB is strongly typed: everything needs a type and it will not try to mix them together. So if you write `SELECT 'Jonathan Harker' + 8;` it will simply refuse with an error: `QueryError: operator '+' cannot be applied to operands of type 'std::str' and 'std::int64'`.

On the other hand, it can use "type inference" to guess the type, and that is what it does here: it knows that we are creating a `str`. We will look at changing types and working with different types soon.

## Links

So now the last thing left to do is to change our `property` in `Person` called `places_visited` to a `link`. Right now, `places_visited` gives us the names we want, but it makes more sense to link `Person` and `City` together. After all, the `City` type has `.name` inside it which is better to link to than rewriting everything inside `Person`. We'll change `Person` to look like this:

```sdl
type Person {
  required property name -> str;
  multi link places_visited -> City;
}
```

We wrote `multi` in front of `link` because one `Person` should be able to link to more than one `City`. The opposite of `multi` is `single`, which only allows one object to link to it. But `single` is the default, so if you just write `link` then EdgeDB will treat it as `single`.

Now when we insert Jonathan Harker, he will be connected to the type `City`. Don't forget that `places_visited` is not `required`, so we can still insert with just his name to create him:

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
};
```

But this would only create a `Person` type connected to the `City` type but with nothing in it. Let's see what's inside:

```edgeql
SELECT Person {
  name,
  places_visited
};
```

Here is the output: `{Object {name: 'Jonathan Harker', places_visited: {}}}`

But we want to have Jonathan be connected to the cities he has traveled to. We'll change `places_visited` when we `INSERT` to `places_visited := City`:

```edgeql
INSERT Person {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

We haven't filtered anything, so it will put all the `City` types in there. Now let's see the places that Jonathan has visited. The code below is almost but not quite what we need:

```edgeql
select Person {
  name,
  places_visited
};
```

Here is the output:

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

Close! But we didn't mention any properties inside `City` so we just got the object id numbers. Now we just need to let EdgeDB know that we want to see the `name` property of the `City` type. To do that, add a colon and then put `name` inside curly brackets.

```edgeql
select Person {
  name,
  places_visited: {
    name
  }
};
```

Success! Now we get the output we wanted:

```
  Object {
    name: 'Jonathan Harker',
    places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}},
  },
```

Of course, Jonathan Harker has been inserted with a connection to every city in the database. Right now we only have three `City` objects, so this is no problem yet. But later on we will have more cities and won't be able to just write `places_visited := City` for all the other characters. For that we will need `FILTER`, which we will learn to use in the next chapter.

[Here is all our code so far up to Chapter 1.](code.md)

<!-- quiz-start -->

## Time to practice

1. Entering the code below returns an error. Try adding one character to make it return `{true}`.

   ```edgeql
   WITH my_name = 'Timothy',
   SELECT my_name != 'Benjamin';
   ```

2. Try inserting a `City` called Constantinople, but now known as İstanbul.
3. Try displaying all the names of the cities in the database. (Hint: you can do it in a single line of code and won't need `{}` to do it)
4. Try selecting all the `City` types along with their `name` and `modern_name` properties, but change `.name` to say `old_name` and change `modern_name` to say `name_now`.
5. Will typing `SelecT City;` produce an error?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan Harker arrives in Romania._
