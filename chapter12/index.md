---
tags: Overloading Functions, Coalescing
---

# 第十二章 - 越来越糟糕

这一章没有给我们的英雄们带来什么好消息。

> 每次人们不听范海辛（Van Helsing）的话，德古拉（Dracula）就会再次闯入露西（Lucy）的房间，每次男同胞们都要献血去救她。德古拉总是化成一团云溜进来，喝露西的血，并在天亮之前溜走。露西变得十分虚弱，以至于她还活着都令人感到惊讶。与此同时，伦菲尔德（Renfield）冲出牢房，用刀袭击了苏厄德医生（Dr. Seward）。他用刀割伤了苏厄德医生，当他看到苏厄德医生的血时，他停了下来并尝试喝下它，嘴里重复着：“血就是生命！血就是命！”。收容所的安保带走了伦菲尔德，苏厄德营生感到十分困惑并试图找到原因。他认为这和其他事件之间存在联系。那天晚上，德古拉操控的一匹狼打破了露西房间的窗户，德古拉得以再次进入……

但是对我们来说有个好消息，那就是我们将继续在本章学习笛卡尔乘积，以及如何重载一个函数。

## 重载函数（Overloading functions）

上一章，我们对一些角色使用了 `fight()` 函数，但大多数角色的 `strength` 是 `{}`。这就是为什么客栈老板（the Innkeeper）打败了德古拉（Dracula），这显然不是真正会发生的事情。

乔纳森·哈克是一个人类，但他仍然是很强壮的。我们将他的力量值设为 5。我们将其视为人类的最大力量，除了有点独特的伦菲尔德（Renfield）。所有其他人的力量都应该在 1 到 5 之间。EdgeDB 有一个名为 `std::rand()` 的随机函数，它会给出一个介于 0.0 和 1.0 之间的 `float64`。还有一个叫做 [round()](https://www.edgedb.com/docs/edgeql/funcops/generic/#function::std::round) 的函数可以对数字进行四舍五入，我们也将使用它，最后将其转换为 `<int16>`。我们的输入如下所示：

```edgeql
SELECT <int16>round(random() * 5);
```

所以现在我们将使用它来更新所有 `strength` 尚为 `{}` 的 `Person`，并赋予它们一个随机的强度值。

```edgeql
WITH random_5 := (SELECT <int16>round(random() * 5))
 # WITH isn't necessary - just making the query prettier

UPDATE Person
FILTER NOT EXISTS .strength
SET {
  strength := random_5
};
```

我们将确保德古拉伯爵获得 20 点力量，因为他是德古拉呀：

```edgeql
UPDATE Vampire
FILTER .name = 'Count Dracula'
SET {
  strength := 20
};
```

现在让我们来 `SELECT Person.strength;` 看看上面的操作是否有效：

```
{3, 3, 3, 2, 3, 2, 2, 2, 3, 3, 3, 3, 4, 1, 5, 10, 4, 4, 20, 4, 4, 4, 4}
```

看起来它奏效了！

所以现在让我们重载 `fight()` 函数。现在它只适用于一个 `Person` 对另一个 `Person`，但在书中所有角色将聚集在一起试图击败德古拉。我们需要重载该函数，以便支持多个角色可以一起战斗。有很多方法可以做到，但我们会选择一种简单的方法：

```sdl
function fight(names: str, one: int16, two: Person) -> str
  using (
    SELECT names ++ ' win!' IF one > two.strength ELSE two.name ++ ' wins!'
  );
```

注意，重载只在函数签名不同的情况下有效。这是我们现在有的两个签名，可以比较一下

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: Person) -> str
```

如果我们试图用 `(Person, Person)` 的输入重载它，它并不会工作，因为两个签名是相同的。EdgeDB 是通过我们给它的输入来判断要使用哪种形式的函数的。

所以现在是相同的函数名称，但我们的输入是一起战斗的人们的名字，及他们的力量和，然后是他们正在战斗试图对抗的 `Person`。

现在乔纳森（Jonathan）和伦菲尔德（Renfield）将要尝试一起对抗德古拉（Dracula）。祝他们好运！

```edgeql
WITH
  jon_and_ren_strength := <int16>(
    SELECT sum(
      (SELECT NPC FILTER .name IN {'Jonathan Harker', 'Renfield'}).strength
    )
  ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

SELECT fight('Jon and Ren', jon_and_ren_strength, dracula);
```

结果还是德古拉赢了：

```
{'Count Dracula wins!'}
```

他们没能赢，那如果是四个人呢？

```edgeql
WITH
  four_people_strength := <int16>(
    SELECT sum(
      (
        SELECT NPC
        FILTER .name IN {'Jonathan Harker', 'Renfield', 'Arthur Holmwood', 'The innkeeper'}
      ).strength
    )
  ),
  dracula := (SELECT Person FILTER .name = 'Count Dracula'),

  SELECT fight('The four people', four_people_strength, dracula);
```

好多了：

```
{'The four people win!'}
```

这就是函数重载的工作原理 —— 只要签名不同，你就可以创建具有相同名称的函数。

你会在许多现有函数中看到重载，例如 [sum](https://www.edgedb.com/docs/edgeql/funcops/set#function::std::sum) 可以接受所有数字类型并返回其总和。[std::to_datetime](https://www.edgedb.com/docs/edgeql/funcops/datetime#function::std::to_datetime) 有着更有趣的重载，支持各种输入来创建一个 `datetime`。

`fight()` 制作起来很有趣，但这种功能更适合在游戏中使用。因此，让我们创建一个我们可能实际会使用到的函数。由于 EdgeQL 是一种查询语言，所以最有用的函数通常是使查询变得更短的函数。

这是一个简单的方法，它告诉我们一个 `Person` 类型的对象是否造访了一个 `Place`：

```sdl
function visited(person: str, city: str) -> bool
  using (
    WITH person := (SELECT Person FILTER .name = person LIMIT 1),
    SELECT city IN person.places_visited.name
  );
```

现在我们的查询要方便得多：

```edgeql-repl
edgedb> SELECT visited('Mina Murray', 'London');
{true}
edgedb> SELECT visited('Mina Murray', 'Bistritz');
{false}
```

多亏了这个函数，即使是更复杂的查询也仍然具有相当的可读性

```edgeql
SELECT(
  'Did Mina visit Bistritz? ' ++ <str>visited('Mina Murray', 'Bistritz'),
  'What about Jonathan and Romania? ' ++ <str>visited('Jonathan Harker', 'Romania')
);
```

打印结果：`{('Did Mina visit Bistritz? false', 'What about Jonathan and Romania? true')}`。

创建函数的文档 [在这里](https://www.edgedb.com/docs/edgeql/ddl/functions#create-function)。你可以看到我们可以使用 SDL 或 DDL 来创建它们，且两者之间没有太大区别。事实上，它们十分相似，唯一的区别是 DDL 需要使用 `CREATE`。换句话说，只需添加 `CREATE` 即可创建函数，而无需进行显式迁移（migration）。例如，这里有一个只是打招呼的函数：

```sdl
function say_hi() -> str
  using ('hi');
```

如果你想立即创建它，只需执行以下操作：

```edgeql
CREATE FUNCTION say_hi() -> str
  USING ('hi');
```

（关键字或用小写字母，没关系的）

当你询问 `DESCRIBE FUNCTION say_hi` 时，你或多或少会看到相同的内容：

```
{'CREATE FUNCTION default::say_hi() ->  std::str USING (\'hi\');'}
```

## 删除函数（Deleting (dropping) functions）

你可以使用关键字 `DROP` 和函数签名来删除函数。不过，你只需指定输入，因为 EdgeDB 在识别一个函数时只查看输入。所以在我们的两个 `fight()` 函数的例子中：

```sdl
fight(one: Person, two: Person) -> str
fight(names: str, one: int16, two: Person) -> str
```

你可以用 `DROP Fight(one: Person, two: Person)` 和 `DROP Fight(names: str, one: int16, two: Person)` 来删除它们。并不需要`-> str` 部分。

## 合并运算符（More about Cartesian products - the coalescing operator）

现在让我们来更多地了解 EdgeDB 中的笛卡尔乘积。你可能会惊讶地发现，即使是单个 `{}` 输入也总是会导致输出 `{}`，但这就是笛卡尔乘积的工作方式。请记住，`{}` 的长度为 0，任何乘以 0 的值也是 0。例如，让我们尝试将以 b 开头的地名和以 f 开头的地名加在一起。

```edgeql
WITH b_places := (SELECT Place FILTER Place.name ILIKE 'b%'),
     f_places := (SELECT Place FILTER Place.name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name;
```

输出是……，如果你没有阅读上一段，结果这可能会令你感到出乎意料。

```
{}
```

！是一个空集。但是搜索以 b 开头的地方，我们明明会得到 `{'Buda-Pesth', 'Bistritz'}`。我们并没有以 f 开头的地方，那么让我们直接用 `++` 连接 `{'Buda-Pesth', 'Bistritz'}` 和 `{}`，看看是否会是同样的效果。

```edgeql
SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
```

你可能以为会同样得到 `{}`。但实际上结果是……

```
error: operator '++' cannot be applied to operands of type 'std::str' and 'anytype'
  ┌─ query:1:8
  │
1 │ SELECT {'Buda-Pesth', 'Bistritz'} ++ {};
  │        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Consider using an explicit type cast or a conversion function.
```

另一个惊喜！不过这很重要：EdgeDB 需要对空集进行强制转换，因为它不会尝试猜测这个空集是什么类型。如果我们给出的只是 `{}`，则无法猜测出其类型，因此 EdgeDB 不会尝试猜测。你可能会猜到数组构造器也是如此，因此 `SELECT [];` 会返回错误：`QueryError: expression returns value of indeterminate type`。


好的，再来一次，这次确保 `{}` 空集是 `str` 类型：

```edgeql-repl
edgedb> SELECT {'Buda-Pesth', 'Bistritz'} ++ <str>{};
{}
```

很好，所以我们已经亲自动手确认了将 `{}` 与另一个集合一起使用（进行连接/相连）总是返回 `{}`。但是如果我们想要以下两点，该怎么做？

- 如果两个字符串均存在则相邻，
- 如果有一个是空集，则返回我拥有的。

换言之，将 `{'Buda-Peth', 'Bistritz'}` 与另一个集合相加时，如何在另一个集合是空时返回原本的 `{'Buda-Peth', 'Bistritz'}`。

为此，我们可以使用 [coalescing operator（合并操作符）](https://www.edgedb.com/docs/edgeql/funcops/set#operator::COALESCE)，写为 `??`。所以对于 `A ?? B` 可以很好地、简单地解释为：

`Evaluate to A for non-empty A, otherwise evaluate to B.`

即如果左侧的项目不为空，它将返回该项目，否则将返回右侧的项目。

这是一个快速示例：

```edgeql-repl
edgedb> SELECT <str>{} ?? 'Count Dracula is now in Whitby';
```

因为我们使用了 `??` 而不是 `++`，所以结果是 `{'Count Dracula 现在在 Whitby'}` 而不是 `{}`。

让我们回到最初的查询，这次使用合并运算符（coalescing operator）：

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     f_places := (SELECT Place FILTER .name ILIKE 'f%'),
SELECT b_places.name ++ ' ' ++ f_places.name
  IF EXISTS b_places.name AND EXISTS f_places.name
  ELSE b_places.name ?? f_places.name;
```

得到返回：

```
{'Buda-Pesth', 'Bistritz'}
```

变得更好了。

但现在回到笛卡尔乘积。请记住，当我们相加或连接集合时，我们要分别处理 _每个集合中的每个项目_。因此，如果我们更改查询为搜索以 b（Buda-Pesth 和 Bistritz）和 m（慕尼黑）开头的地方：
But now back to Cartesian products. Remember, when we add or concatenate sets we are working with _every item in each set_ separately. So if we change the query to search for places that start with b (Buda-Pesth and Bistritz) and m (Munich):

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER .name ILIKE 'm%'),
SELECT b_places.name ++ ' ' ++ m_places.name
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

然后我们会得到这样的结果：

```
{'Buda-Pesth Munich', 'Bistritz Munich'}
```

而不是 `{'Buda-Peth, Bistritz, Munich'}`。

让我们在引入 `array_agg` 和 `array_join` 两个新函数的同时进行更多实验。以下是他们的功能：

- [array_agg](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_agg)，将集合转换为数组（“聚合”它们）。
- [array_join](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_join) 将数组转换为单个字符串。

现在来让我们试一下：

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER .name ILIKE 'm%'),
SELECT array_join(array_agg(b_places.name), ', ') ++ ', ' ++
  array_join(array_agg(m_places.name), ', ')
  IF EXISTS b_places.name AND EXISTS m_places.name
  ELSE b_places.name ?? m_places.name;
```

看起来还不错：输出是 `{'Buda-Pesth, Bistritz, Munich'}`。但是有一个小问题：

- 如果两个集合都不为空，我们会得到一个带逗号的字符串，
- 否则我们得到一个字符串的集合。

所以这不是很健壮。此外，现在的查询有点难以阅读。

最好的方法实际上是最简单的：只需要 `UNION` 集合。

```edgeql
WITH b_places := (SELECT Place FILTER .name ILIKE 'b%'),
     m_places := (SELECT Place FILTER Place.name ILIKE 'm%'),
     both_places := b_places UNION m_places,
SELECT both_places.name;
```

最终，输出是：`{'Buda-Pesth', 'Bistritz', 'Munich'}`

现在有了这个更强大的查询，我们就可以在任何情况下使用它了，而不需要担心如果我们选了一个像 x 这样的字母作为城市开头字母的过滤条件会得到 {}。让我们看看所有包含 k 或 e 的地方：

```edgeql
WITH has_k := (SELECT Place FILTER .name ILIKE '%k%'),
     has_e := (SELECT Place FILTER .name ILIKE '%e%'),
     has_both := has_k UNION has_e,
SELECT has_both.name;
```

输出结果是:

```
{'Slovakia', 'Buda-Pesth', 'Castle Dracula'}
```

类似地，如果你认为比较的其中一侧可能是空集，则在进行比较时可以使用 `?=` 代替 `=` 以及使用 `?!=` 代替 `!=`。因此你可以写一个这样的查询：

```edgeql
WITH cities1 := {'Slovakia', 'Buda-Pesth', 'Castle Dracula'},
     cities2 := <str>{}, # Don't forget to cast to <str>
SELECT cities1 ?= cities2;
```

输出结果是：

```
{
  false,
  false,
  false
}
```

而不是 `{}` 。此外，如果你使用`?=`，则两个空集被视为相等。所以这个查询：

```edgeql
SELECT Vampire.lover.name ?= Crewman.name;
```

将返回 `{true}`。（因为德古拉没有情人，船员也没有名字，所以双方返回的都是类型为 `str` 的空集。）

[→ 点击这里查看第 12 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 考虑下面这两个函数。EdgeDB 会接受第二个吗？

   第一个函数：

   ```sdl
   function gives_number(input: int64) -> int64
     using(input);
   ```

   第二个函数：

   ```sdl
   function gives_number(input: int64) -> int32
     using(<int32>input);
   ```

2. 那么下面两个函数呢？EdgeDB 会接受第二个吗？

   第一个函数：

   ```sdl
   function make64(input: int16) -> int64
     using(input);
   ```

   第二个函数：

   ```sdl
   function make64(input: int32) -> int64
     using(input);
   ```

3. `SELECT {} ?? {3, 4} ?? {5, 6};` 能工作吗？

4. `SELECT <int64>{} ?? <int64>{} ?? {1, 2}` 能工作吗

5. `SELECT array_join(array_agg(Person.name));` 在尝试获得一个含有所有人姓名的字符串，但它不能工作，问题出在哪里？

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _其中一名男子献血给露西试图拯救她。就够了吗？_
