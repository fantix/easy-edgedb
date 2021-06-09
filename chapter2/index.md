---
tags: Scalar Types, Abstract Types, Filter
leadImage: illustration_02.jpg
---

# 第二章 - 在比斯特里茨的酒店

我们继续阅读这个故事，并思考哪些信息我们需要存入数据库。重要的信息以粗体显示：

> 乔纳森·哈克（Jonathan Harker）在 **比斯特里茨（Bistritz）** 发现了一家酒店, 叫做 **金克朗酒店（Golden Krone Hotel）**。他在酒店里收到一封来自德拉库拉的欢迎信，信中说明德拉库拉正在 **城堡（castle）** 里等他。乔森纳·哈克（Jonathan Harker）明天必须搭乘 **马车（horse-driven carriage）** 才能到达那里。我们也看到乔纳森·哈克（Jonathan Harker）来自 **伦敦（London）**。金克朗酒店（Golden Krone Hotel）的老板似乎很害怕德古拉。他不想让乔纳森（Jonathan）离开并表明前往城堡会很危险，但乔纳森（Jonathan）并没有听进去。一位老太太给了乔纳森（Jonathan）一个金色的十字架，并说这会保护他。乔纳森（Jonathan）感到尴尬，但认为这可能是出于礼貌，他并不知道之后这会对他有多大的帮助。

现在我们开始看一下关于这座城市的一些细节。通过阅读这个故事，我们看到我们可以添加另一个属性给 `City`，我们叫它 `important_places`。比如可以去像 **金克朗酒店（Golden Krone Hotel）** 这样的地方。我们尚不确定这些地方是否将拥有属于他们自己的类型，因此我们只是定义它为一个字符串数组，像这样：`property important_places -> array<str>;` 我们可以把这些重要地点的名字放进去，也许之后还会发展出更多的内容。现在它看起来像这样：

```sdl
type City {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
```

现在我们对 Bistritz 的原始插入将如下所示：

```edgeql
INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};
```

## 枚举、标量类型和扩展（Enums, scalar types, and extending）

到目前位置，我们在书中有提到两种交通工具：火车和马车。这本书以 1887 年为背景，我们的游戏将让角色使用当年可用的交通工具。这里的 `enum`（枚举）可能是最好的选择，因为 `enum` 是在选项之间做出一个选择。枚举的变量应该用大写驼峰式（UpperCamelCase）进行书写。

这里我们第一次看到单词 `scalar`：这是一个“标量类型”（`scalar type`），因为它一次只保存一个值。其他类型（`City`, `Person`）是“对象类型”（`object types`）因为他们能够同时保存多个值。

另一个我们第一次看到的关键词是 `extending`：它意味着以一个类型作为基础并扩展它。这使你对你想要扩展的类型有更多的权利，可以添加更多选项。我们将按如下来写 `Transport` 类型：

```sdl
scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;
```

你是否留意到 `scalar type` 是以一个分号结尾的，而其他类型并非如此？这是因为其他类型有一个 `{}` 以构成一个完整的表达式。但是这里的单行代码我们并没有 `{}`，所以在这里我们需要用分号来说明表达式的结束。

这个 `Transport` 类型将被用于我们游戏中的玩家角色，而不是书中的人物（他们的故事和选择已成定局）。这意味着我们需要一个 `PC` 类型和一个 `NPC` 类型，但我们的 `Person` 类型也应该保留 - 我们可以将它用作两者的基本类型。为此，我们可以让 `Person` 成为一个 `abstract type` 而不仅仅是一个 `type`。然后有了这个抽象类型，我们可以对其他 `PC` 和 `NPC` 类型使用关键字 `extending`。

所以现在这部分架构看起来像这样：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
}

type PC extending Person {
  required property transport -> Transport;
}

type NPC extending Person {
}
```

现在书中的角色将是 `NPC`（非玩家角色），而 `PC` 是在考虑我们的游戏的情况下设定的。因为 `Person` 现在是一个抽象类型，我们不能再对其进行直接的插入。如果你尝试执行 `INSERT Person {name := 'Mr. HasAName'};`，将会收到错误提示：

```
error: cannot insert into abstract object type 'default::Person'
  ┌─ query:1:8
  │
1 │ INSERT Person {
  │        ^^^^^^^ error
```

没关系 —— 只要将 `Person` 改为 `NPC`，它就可以工作了。

此外，`SELECT` 一个抽象类型是没有问题的 —— 它将会选择出所有从它扩展出的类型。

让我们也试验一下玩家角色。我们创建一个名叫 Emil Sinclair 的人，他开始乘坐马车旅行。我们也将 `City` 给他，于是他也拥有了三个造访过的城市。

```edgeql
INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := <Transport>'HorseDrawnCarriage',
};
```

`places_visited := City` 是对 `places_visited := (SELECT City)` 的简写 —— 你不是必须每次都输入 `SELECT` 部分。

请注意，我们并没有只是写了 `HorseDrawnCarriage`，我们必须选择枚举类型 `Transport` 并选择其中一个枚举值。`<>` 尖括号做 _casting_，意思是将一种类型转换为另一种类型。EdgeDB 不会自行尝试将一种类型更改为另一种类型，除非你要求它进行转换。这就是为什么下面的语句不会给我们 `true` 的原因：

```edgeql
SELECT 'Feet' IS Transport;
```

我们将得到一个 `{false}` 的输出结果，因为 'Feet' 仅仅是一个 `str`。但是下面将会返回 `true`：

```edgeql
SELECT <Transport>'Feet' IS Transport;
```

然后我们如愿得到 `{true}`。

如果需要，你可以一次性转换多次。下面这个例子不是你需要做的，只是为了展示如果你愿意，你可以如何一遍又一遍地做类型转换：

```edgeql
SELECT <str><int64><str><int32>50 is str;
```

这个也会返回我们 `{true}`，因为我们所做的只是询问它是否是一个 `str`，且它确实是。

类型转换从右往左执行，最后的转换是在最左侧。因此，`<str><int64><str><int32>50` 意味着“50 先变成了 int32，再变成了 str，又变成了 int64，最后又变成了 str。

此外，需要注意类型转换仅适用于标量类型 `scalar type`：用户创建的对象类型，如 `City` 和 `Person` 都太复杂，并不能简单地相互转换。

## Filter

Finally, let's learn how to `FILTER` before we're done Chapter 2. You can use `FILTER` after the curly brackets in `SELECT` to only show certain results. Let's `FILTER` to only show `Person` types that have the name 'Emil Sinclair':

```edgeql
SELECT Person {
  name,
  places_visited: {name},
} FILTER .name = 'Emil Sinclair';
```

`FILTER .name` is short for `FILTER Person.name`. You can write `FILTER Person.name` too if you want - it's the same thing.

The output is this:

```
{Object {name: 'Emil Sinclair', places_visited: {Object {name: 'Munich'}, Object {name: 'Buda-Pesth'}, Object {name: 'Bistritz'}}}}
```

Let's filter the cities now. One flexible way to search is with `LIKE` or `ILIKE` to match on parts of a string.

- `LIKE` is case-sensitive: "Bistritz" matches "Bistritz" but "bistritz" does not.
- `ILIKE` is not case-sensitive (the I in ILIKE means **insensitive**), so "Bistritz" matches "BiStRitz", "bisTRITz", etc.

You can also add `%` on the left and/or right which means match anything before or after. Here are some examples with the matched part **in bold**:

- `LIKE Bistr%` matches "**Bistr**itz" (but not "bistritz"),
- `ILIKE '%IsTRiT%'` matches "B**istrit**z",
- `LIKE %athan Harker` matches "Jon**athan Harker**",
- `ILIKE %n h%` matches "Jonatha**n H**arker".

Let's `FILTER` to get all the cities that start with a capital B. That means we'll need `LIKE` because it's case-sensitive:

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name LIKE 'B%';
```

Here is the result:

```
  Object {name: 'Buda-Pesth', modern_name: 'Budapest'},
  Object {name: 'Bistritz', modern_name: 'Bistrița'},
```

You can also index a string with `[]` square brackets, starting at 0. For example, the indexes in the string 'Jonathan' look like this:

```
J o n a t h a n
0 1 2 3 4 5 6 7
```

So `'Jonathan'[0]` is 'J' and `'Jonathan'[4]` is 't'.

Let's try it:

```edgeql
SELECT City {
  name,
  modern_name,
} FILTER .name[0]; = 'B'; # First character must be 'B'
```

That gives the same result. Careful though: if you set the number too high then it will try to search outside of the string, which is an error. If we change 0 to 18 (`FILTER .name[18]; = 'B';`), we'll get this:

```
ERROR: InvalidValueError: string index 18 is out of bounds
```

Plus, if you have any `City` types with a name of `''`, even a search for index 0 will cause an error. 

You can also slice a string to get a piece of it. Because 'Jonathan' starts at zero, its index values look like this:

```
|J|o|n|a|t|h|a|n|
0 1 2 3 4 5 6 7 8
```

It's 8 characters long, so it fits entirely between 0 and 8. If you take a "slice" of it between indexes 2 and 5, you get 'nat' (`'Jonathan'[2:5]` = 'nat'), because it starts at 2 and goes *up to* 5 - but not including index 5. It's sort of like when you phone your friend to tell them that you're 'at their house': you're not telling them that you're inside it.

Negative index values are counted from the end of 'Jonathan', which is 8, so -1 corresponds to `8 - 1` (= 7), etc.

So what if you want to make sure that you won't get an error with an index number that might be too high? Here you can use `LIKE` or `ILIKE` with an empty parameter, because it will just give an empty set: `{}` instead of an error. `LIKE` and `ILIKE` are safer than indexing if there is a chance of having data that is too short in a property. There is a small lesson to be had here:

- "no data" in Edgedb is shown as an empty set: `{}`
- `""` (an empty string) is actually data.

Keeping that in mind helps you understand the behaviour between the two.


Finally, did you notice that we wrote a comment with `#` just now? Comments in EdgeDB are simple: anything to the right of `#` on a line gets ignored.

So this:

```edgeql
SELECT 1887#0503 is the first day of the book Dracula when...
;
```

returns `{1887}`.

[Here is all our code so far up to Chapter 2.](code.md)

<!-- quiz-start -->

## Time to practice

1. Change the following `SELECT` to display `{100}` by casting: `SELECT '99' + '1'`;
2. Select all the `City` types that start with 'Mu' (case sensitive).
3. Select the third letter (i.e. index number 2) of the name of every `NPC`.
4. Imagine an abstract type called `HasAString`:

   ```sdl
   abstract type HasAString {
     property string -> str
   };
   ```

   How would you change the `Person` type to extend `HasAString`?

5. This query only shows the id numbers of the places visited. How do you show their name?

   ```edgeql
   SELECT Person {
     places_visited
   };
   ```

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan gets into the carriage and travels into the cold mountains._
