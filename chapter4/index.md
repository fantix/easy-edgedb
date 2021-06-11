---
tags: Local Time, Advanced Filtering
leadImage: illustration_04.jpg
---

# 第四章 - "德拉库拉伯爵真是个奇怪的人"

> 乔纳森·哈克（Jonathan Harker）起晚了，独自一人在城堡里。夜幕降临后，德拉库拉出现了，他们聊了**一个通宵（through the night）**。德拉库拉（Dracula）正在制定搬到伦敦的计划，乔纳森（Jonathan）给了他一些关于买房的建议。乔纳森告诉德拉库拉，一个叫 Carfax 的大房子是一个值得购买的好房子。它很大且安静。它附近有一家疯人院，但又不至于太近。德拉库拉喜欢这个主意。然后他告诉乔纳森不要进入城堡中任何上了锁的房间，因为很危险。乔纳森发现已经快到早上了 —— 他们又聊了整整一夜。德拉库拉突然站起来说他必须走了，然后离开了房间。乔纳森想着回到了伦敦的**米娜（Mina）** ，等他回去他们即将完婚。他开始觉得德拉库拉和他的城堡有些问题。说真的，其他人在哪里？

首先，让我们创建乔纳森的女朋友 —— 米娜·默里（Mina Murray）。同时我们还将增加在 `Person` 类型的结构中增加一个新的指向 `Person` 类型的链接，并命名为 `lover`：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  link lover -> Person;
}
```

这样我就可以把乔纳森和米娜联系在一起了。我们假设一个人只能有一个 `lover`，所以它是一个 `single link`，但我们可以只写 `link`。

米娜目前在伦敦，且我们不知道她是否去过其他什么地方。所以让我们先快速插入一条数据以创建城市伦敦。这再简单不过了：

```edgeql
INSERT City {
    name := 'London',
};
```

为了记录米娜造访的城市有伦敦市，我们只需要做一个快速的查询：`SELECT City FILTER .name = 'London'`。这将为她提供与 `.name = 'London'` 匹配的 `City`，如果该城市不存在，也不会给出错误：它只会返回一个 `{}` 空集。

## 分离，限制和存在（DETACHED, LIMIT, and EXISTS）

我们创建一个名为“Mina Murray”的 NPC，关于其 `lover` 的赋值部分稍微复杂一些：

```edgeql
INSERT NPC {
  name := 'Mina Murray',
  lover := (SELECT DETACHED NPC 
    FILTER .name = 'Jonathan Harker' 
    LIMIT 1),
  places_visited := (SELECT City FILTER .name = 'London'),
};
```

这里，你可能已经注意到了两件事：

- `DETACHED`：这是因为我们在 `NPC` 类型的 `INSERT` 内想要链接到另一个相同类型的 `NPC`。我们需要添加 `DETACHED` 来指明此处我们正在谈论的是泛指的 `NPC`，而不是现在正在插入的这个 `NPC`。
- `LIMIT 1`：这是因为该链接是 `single link`。 EdgeDB 不知道执行 `SELECT DETACHED NPC FILTER .name = 'Jonathan Harker'` 会得到多少结果，可能有 2 个或 3 个或更多的 `Jonathan Harkers`。为了保证我们只创建一个 `single link`，我们使用 `LIMIT 1`。

现在我们想查询一下谁是单身，谁不是。我们可以在其中创建一个新的变量并用 `:=` 来赋予它一个判断公式。首先先来看一下下面这个常规的查询：

```edgeql
SELECT Person {
  name,
  lover: {
    name
  }
};
```

执行后得到输出：

```
{
  Object {name: 'Jonathan Harker', lover: {}},
  Object {name: 'The innkeeper', lover: {}},
  Object {name: 'Mina Murray', lover: Object {name: 'Jonathan Harker'}},
  Object {name: 'Count Dracula', lover: {}},
}
```

好的，所以米娜·默里（Mina Murray）有一个爱人，但乔纳森·哈克（Jonathan Harker）还没有，因为我们之前创建它的时候还没有 `lover` 这个属性。我们将会在后续的第 6、14、15章学习一些技巧来处理这种情况。这里我们就先暂时保持乔纳森·哈克的 `link lover` 是  `{}`。

回到查询语句：如果我们只是想根据角色是否有爱人来返回 `true` 或 `false` 怎么办？我们可以使用 `EXISTS` 在查询语句中添加一个 “computable”（可计算的数据，类似 `NOT EXISTS Person.lover`）。如下所示，如果 `Person.lover` 返回的是一个有内容的集合，则`EXISTS` 将返回 `true`；如果它返回的是 `{}`（即什么都没有），则 `EXISTS` 返回`false`。这再次表明 EdgeDB 中没有 null。由此，我们调整查询语句为：

```edgeql
SELECT Person {
  name,
  is_single := NOT EXISTS Person.lover,
};
```

这次打印出的结果是：

```
  Object {name: 'Count Dracula', is_single: true},
  Object {name: 'The innkeeper', is_single: true},
  Object {name: 'Mina Murray', is_single: false},
  Object {name: 'Jonathan Harker', is_single: true},
  Object {name: 'Emil Sinclair', is_single: true},
```

这也说明了为什么抽象类型很有用。在这里，我们快速搜索了 `Person` 中来自 `Vampire`、`PC` 和 `NPC` 的所有数据，因为它们都来自 `abstract type Person`。

我们也可以将可计算数据（“computables”）放在类型本身中。如下所示，我们把 `NOT EXISTS .lover` 放到了 `Person` 类型定义中：

```sdl
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  property lover -> Person;
  property is_single := NOT EXISTS .lover;
}
```

不过我们接下来并不会在这个类型定义中保留 `is_single`，这里只是示意你还可以如何做。

您可能对可计算数据（“computables”）在后端数据库中的表示方式感到好奇。它们很有趣，因为它们 [不会出现在实际数据库中](https://www.edgedb.com/docs/datamodel/computables)，他们仅在你查询时出现。当然你不必指定类型，因为这由可计算数据自身决定。比如，当你的查询带有一个快速可计算公式，类似 `SELECT country_name := 'Romania'` 时，每次执行查询时 EdgeDB 都会计算 `country_name` ，且类型被确定为字符串。当可计算数据（“computables”）放在类型本身中，所做的是事情是一样的。类型定义中的计算属性做的事情其实是一样的。但无论如何，计算属性用起来同常规的属性或者链接并无二致，因为计算属性的公式在定义类型的时候就写死了。换言之，计算属性虽底层实现不同，但对使用者来说都是一样的属性。

## 报时的方法（Ways to tell time）

现在我们来学习时间，这也是十分重要的。请记住，吸血鬼只能在夜幕降临后出去（因为他们害怕阳光）。

乔纳森·哈克（Jonathan Harker）正处的罗马尼亚平均的日出时间是 7 am，日落时间是 7 pm。随着季节不同会有所变化，但为了简单起见，我们将仅使用早上 7 点和晚上 7 点来决定是白天还是黑夜。

EdgeDB 使用两种主要的时间类型。

- `std::datetime`：这是非常精确的并且总是有一个时区。`datetime` 中的时间使用 ISO 8601 标准。
- `cal::local_datetime`：这个不会考虑时区。

还有两个与 `cal::local_datetime` 几乎相同的时间类型：

- `cal::local_time`：当你只需要知道一天中的时间时；
- `cal::local_date`：当您只需要知道月、日和年时。

我们先从 `cal::local_time` 开始。

`cal::local_time` 是很容易创建的，因为你可以从“HH:MM:SS”格式的`str`转换到它：

```edgeql
SELECT <cal::local_time>('15:44:56');
```

执行后输出：

```
{<cal::local_time>'15:44:56'}
```

我们假设我们的游戏故事中有一个时钟，它以 `str` 的形式给出时间，就像上面例子中的 '15:44:56'。让我们来创建一个 `Time` 类型，这会对后续很有帮助：

```sdl
type Time {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
}
```

`.date[0:2]` 是 ["切片（slicing）"](https://www.edgedb.com/docs/edgeql/funcops/array#operator::ARRAYSLICE) 的一个例子。[0:2] 表示从索引 0（第一个索引）开始，在索引 2_之前_ 停止，即索引 0 和 1。这也说明当你要将 `str` 转换为 `cal::local_time`时，您需要用两个数字来表示小时（例如 09 可以，但 9 不行）。

即如下语句是无法工作的：

```edgeql
SELECT <cal::local_time>'9:55:05';
```

它将会给出错误：

```
ERROR: InvalidValueError: invalid input syntax for type cal::local_time: '9:55:05'
```

因此，我们确信从索引 0 到 2 的切片将为我们提供两个表示一天中的小时的数字。

现在使用这个 `Time` 类型，我们可以通过执行以下操作来获取小时。首先插入时间：

```edgeql
INSERT Time {
    date := '09:55:05',
};
```

然后我们可以 `SELECT` 我们的 `Time` 对象和里面的所有东西：

```edgeql
SELECT Time {
  date,
  local_time,
  hour,
};
```

执行后，我们得到了期望的输出，它展示了所有，包括小时：

`{Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09'}}`.

最后，我们可以在 `Time` 类型中添加一些逻辑来查看吸血鬼是醒着还是睡着了。我们可以使用一个 `enum`，但为了简单起见，我们将它设为一个 `str`。

```sdl
type Time {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 
    ELSE 'awake';
}
```

因此，`awake` 是如下这样计算的：

- 首先 EdgeDB 会查看取到的小时的数值是否大于 7 且小于 19（晚上 7 点）。但是这里用数字进行比较比与字符串进行比较要好，因此我们编写`<int16>.hour` 而不是`.hour`，这样它就可以通过强制转换用数字与数字进行比较。
- 然后 EdgeDB 会基于比较结果给出一个字符串来说明现在的状态是“睡着（'asleep'）”或“醒着（'awake'）”。

现在，如果我们对所有属性进行 `SELECT`，我们将得到：

`Object {date: '09:55:05', local_time: <cal::local_time>'09:55:05', hour: '09', awake: 'asleep'}`

关于 `ELSE` 的另一个注意事项：你可以在 `(result) IF (condition) ELSE` 中多次使用 `ELSE`。下面是一个例子：

```
property awake := 'just waking up' IF <int16>.hour = 19 ELSE
                  'going to bed' IF <int16>.hour = 6 ELSE
                  'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 
                  'awake';
```

## SELECT while you INSERT

Back in Chapter 3, we learned how to select while deleting at the same time. You can do the same thing with `INSERT` by enclosing it in brackets and then selecting that, same as with any other `SELECT`. Because when we insert a new `Time`, all we get is a `uuid`:

```edgeql
INSERT Time {
  date := '22:44:10'
};
```

The output is just something like this: `{Object {id: 528941b8-f638-11ea-acc7-2fbb84b361f8}}`

So let's wrap the whole entry in `SELECT ()` so we can display its properties as we insert it. Because it's enclosed in brackets, EdgeDB will do that operation first, and then give it for us to select and do a normal query. Besides the properties to display, we can also add a computable while we are at it. Let's give it a try:

```edgeql
SELECT ( # Start a selection
  INSERT Time { # Put the insert inside it
    date := '22:44:10'
  }
) # The bracket finishes the selection
  { # Now just choose the properties we want
    date,
    hour,
    awake,
    double_hour := <int16>.hour * 2
  };
```

Now the output is more meaningful to us: `{Object {date: '22.44.10', hour: '22', awake: 'awake', double_hour: 44}}` We know the date and the hour, we can see that vampires are awake, and even make a computable from the object we just entered.

[Here is all our code so far up to Chapter 4.](code.md)

<!-- quiz-start -->

## Time to practice

1. This insert is not working.

   ```edgeql
   INSERT NPC {
     name := 'I Love Mina',
     lover := (SELECT NPC FILTER .name LIKE '%Mina%' LIMIT 1)
   };
   ```

   The error is: `invalid reference to default::NPC: self-referencing INSERTs are not allowed`. What keyword can we use to make this insert work?

   Bonus: there is another method we could use too to make it work without the keyword. Can you think of another way?

2. How would you display up to 2 `Person` types (and their `name` property) whose names include the letter `a`?

3. How would you display all the `Person` types (and their names) that have never visited anywhere?

   Hint: all the `Person` types for which `.places_visited` returns `{}`.

4. Imagine that you have the following `cal::local_time` type:

   ```edgeql
   SELECT has_nine_in_it := <cal::local_time>'09:09:09';
   ```

   This displays `{<cal::local_time>'09:09:09'}` but instead you want to display {true} if it has a 9 and {false} otherwise. How could you do that?

5. We are inserting a character called The Innkeeper's Son:

   ```edgeql
   INSERT NPC {
     name := "The Innkeeper's Son",
     age := 10
   };
   ```

   How would you `SELECT` this insert at the same time to display the `name`, `age`, and `age_ten_years_later` that is made from `age` plus 10?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _Jonathan's curiosity gets the better of him. What's inside this castle?_
