---
tags: Filtering On Insert, Json
---

# 第六章 - 乔纳森还是逃不掉

> 女吸血鬼就在乔纳森的身边，且他无法动弹。突然，德拉库拉跑进房间，告诉女人们离开：“晚些时候你们可以拥有他，但今晚不行！”。女吸血鬼们听从了。乔纳森在他的床上醒来，感觉就像做了一场噩梦……但他看到他的衣服被叠好了，他知道这不会仅仅是一个梦。第二天会有一些来自斯洛伐克的游客造访城堡，所以乔纳森想到了个主意。他写了两封信，一封给米娜，一封给他的老板。他付了来访者一些钱，请求他们为他寄信。但是德拉库拉发现了这些信件，他很生气。他在乔纳森面前烧掉了它们，并告诉他不许再这样做了。乔纳森仍然被困在城堡里，德拉库拉知道乔纳森试图欺骗他从而逃跑。

## 插入时过滤集合（Filtering on sets when doing an insert）

本章中没有太多关于类型的新内容，所以让我们来看看如何改进我们的架构（schema）。现在乔纳森·哈克（Jonathan Harker）的插入语句仍然是这样的：

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```

当我们只有城市时，这是没问题的。但现在我们有了 `Place` 和 `Country` 两个类型。首先，我们先插入另外两个 `Country` 类型的对象以增加多样性：

```edgeql
INSERT Country {
  name := 'France'
};
INSERT Country {
  name := 'Slovakia'
};
```

（在第 9 章中，我们将学习如何只用一个 `INSERT` 来做到这一点！）

然后我们为既不是城市也不是国家的地方创建一个名为 `OtherPlace` 的新类型。这很容易做到：`type OtherPlace extending Place;`。

然后我们插入我们的第一个 `OtherPlace`：

```edgeql
INSERT OtherPlace {
  name := 'Castle Dracula'
};
```

这为我们提供了大量来自 `Place` 类型的对象，而不是 `City` 类型。

回到乔纳森：在我们的数据库中，他去过了四个城市、一个国家和一个 `OtherPlace`……但他没有去过斯洛伐克（Slovakia）或法国（France），所以我们不能直接用 `places_visited := SELECT Place` 对其进行插入。但我们可以根据他访问过的地方的名称对 `Place` 进行过滤。像这样：

```edgeql
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name IN {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};
```

你会留意到我们只是使用 `{}` 在集合中写入了这些地方的名称，我们不需要使用带有 `[]` 的数组来执行此操作。（顺便说一下，这称为 [set 构造函数](https://www.edgedb.com/docs/edgeql/expressions/overview/#set-constructor)。）

现在如果乔纳森逃离德拉库拉城堡并到了一个新地方怎么办？让我们假装他逃跑了并逃到了斯洛伐克（Slovakia）。当然，我们修改他的 `INSERT`，在造访点的名字集合里加上 `'Slovakia'`。但是我们如何才能做一次快速的更新呢？对此，我们有关键字 `UPDATE` 和 `SET`。`UPDATE` 要更新的类型，`SET` 我们要更改的部分。像这样：

```edgeql
UPDATE NPC
FILTER .name = 'Jonathan Harker'
SET {
  places_visited += (SELECT Place FILTER .name = 'Slovakia')
};
```

如果语句执行成功了，EdgeDB 将返回被成功更新对象的 ID 们。在上面的例子里，仅仅会返回一个 ID：

```
{
  Object { id: <uuid>"6f436006-3f65-11eb-b6de-a3e7cc8efd4f" }
}
```

如果我们写了类似 `FILTER .name = 'SLLLlovakia'` 的东西，那么 EdgeDB 会返回 `{}`，让我们知道没有任何可匹配的。或者更准确地说：顶级对象在 `FILTER .name = 'Jonathan Harker'` 上得到匹配，但 `places_visited` 并没有得到更新，因为 `FILTER` 里没有获得任何可匹配的。

由于乔纳森（Jonathan）还没有访问过斯洛伐克（Slovakia），我们现在可以使用 `-=` 代替 `+=` 并使用相同的 `UPDATE` 语法来删除它。

现在我们了解到了在`SET` 之后可以使用的 [所有三个运算符](https://www.edgedb.com/docs/edgeql/statements/update) ：`:=`、`+=` 和`-=`。 

让我们来做另一个更新。还记得这个吗？
Let's do another update. Remember this?

```edgeql
SELECT Person {
  name,
  lover
} FILTER .name = 'Jonathan Harker';
```

米娜·默里（Mina Murray）有乔纳森·哈克（Jonathan Harker）作为她的 `lover`，但因为我们先插入了乔纳森，所以乔纳森没有 `lover`。我们现在可以改变它：

```edgeql
UPDATE Person FILTER .name = 'Jonathan Harker'
SET {
  lover := (SELECT Person FILTER .name = 'Mina Murray' LIMIT 1)
};
```

现在 Jonathan 的 `link lover` 终于显示了 Mina 而不再是空的 `{}`。

当然，如果你在没有 `FILTER` 的情况下使用了 `UPDATE`，它将对所有类型进行相同的更改。例如，下面的更新将在数据库中为每个 `Person` 类型的对象的 `places_visited` 存入所有每一个 `Place`：

```edgeql
UPDATE Person
SET {
  places_visited := Place
};
```

## 用 ++ 连接（Concatenation with ++）

另一种运算符是`++`，它执行连接（连接在一起）而不是做“相加”。

你可以像这样：`SELECT 'My name is ' ++ 'Jonathan Harker';` 做简单的操作，得到结果 `{'My name is Jonathan Harker'}`。或者你可以做更复杂的连接，只要你继续将字符串连接到字符串：

```edgeql
SELECT 'A character from the book: ' ++ (SELECT NPC.name) ++ ', who is not ' ++ (SELECT Vampire.name);
```

这会打印出：

```
{
  'A character from the book: Jonathan Harker, who is not Count Dracula',
  'A character from the book: The innkeeper, who is not Count Dracula',
  'A character from the book: Mina Murray, who is not Count Dracula',
}
```

（这个连接运算符也适用于数组，会将它们合并放入到一个数组中。所以执行 `SELECT ['I', 'am'] ++ ['Jonathan', 'Harker'];` 的结果是 `{['I', 'am', 'Jonathan', 'Harker']}`。）

让我们也来更改一下 `Vampire` 类型，使其可以链接指向 `MinorVampire`。你应该记得德拉库拉伯爵是唯一一个真正的吸血鬼，而其他吸血鬼都是 `MinorVampire` 类型。这意味着我们需要一个 `multi link`：

```sdl
type Vampire extending Person {
  property age -> int16;
  multi link slaves -> MinorVampire;
}
```

然后我们可以在插入德拉库拉伯爵（Count Dracula）的信息的同时 `INSERT` `MinorVampire` 类型的对象。但首先让我们先从 `MinorVampire` 中删除`link master`，因为我们不希望两个对象相互链接。原因有二：

- 当我们声明一个 `Vampire` 时，它有 `slaves` 指向 `MinorVampire`，但如果还没有 `MinorVampire`，那么它将是空的：{}。如果我们首先声明`MinorVampire` 类型，它有一个 `master` 指向 `Vampire`，但还没有声明 `Vampire`，那么他们的 `master`（一个 `required link`）将不存在。
- 如果这两种类型相互链接，我们将无法在需要时删除它们。会给出如下的错误：

```
ERROR: ConstraintViolationError: deletion of default::Vampire (cc5ee436-fa23-11ea-85e0-e78b548f5a59) is prohibited by link target policy

DETAILS: Object is still referenced in link master of default::MinorVampire (cc87c78e-fa23-11ea-85e0-8f5149329e3a).
```

因此，首先我们简单地将 `MinorVampire` 更改为 `Person` 的扩展类型：

```sdl
type MinorVampire extending Person {
}
```

然后我们在创建德拉库拉伯爵时，也一同创建她们，如下所示：

```edgeql
INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (INSERT MinorVampire {
      name := 'Woman 1',
    }),
    (INSERT MinorVampire {
      name := 'Woman 2',
    }),
    (INSERT MinorVampire {
      name := 'Woman 3',
    }),
  }
};
```

请注意两件事：（1）我们在 `slaves` 后面使用了 `{}`，并把所有要插入的 `INSERT` 都放到了这个集合；（2）每一个 `INSERT` 都放在了小括号 `()` 里去捕捉插入。

现在我们不必先插入 `MinorVampire` 类型然后再过滤并更新给 `Vampire`：我们可以将她们与德拉库拉（Dracula）放在一起插入。

然后当我们像下面这样 `select Vampire` 时：

```edgeql
SELECT Vampire {
  name,
  slaves: {name}
};
```

由此我们得到想要的输出结果，将德拉库拉（Dracula）和女吸血鬼们都显示了出来：

```
Object {
  name: 'Count Dracula',
  slaves: {Object {name: 'Woman 1'}, Object {name: 'Woman 2'}, Object {name: 'Woman 3'}},
},
```

This might make you wonder: what if we do want two-way links? There's actually a very convenient way to do it (it's called a **backward link**), but we won't look at it until Chapters 14 and 15. If you're really curious you can skip to those chapters but there's a lot more to learn before then.

## Just type \<json> to generate json

What do we do if we want the same output in json? It couldn't be easier: just cast using `<json>`. Any type in EdgeDB (except `bytes`) can be cast to json this easily:

```edgeql
SELECT <json>Vampire {
      # <json> is the only difference from the SELECT above
  name,
  slaves: {name}
};
```

The output is:

```
{
  "{\"name\": \"Count Dracula\", \"slaves\": [{\"name\": \"Woman 1\"}, {\"name\": \"Woman 2\"}, {\"name\": \"Woman 3\"}]}",
}
```

## Converting back from JSON

So what about the other way around, namely JSON to an EdgeDB type? You can do this too, but remember to think about the JSON type that you are giving to cast. The EdgeDB philosophy is that casts should be symmetrical: a type cast into JSON should only be cast back into that type. For example, here is the first date in the book Dracula as a string, then cast to JSON and then into a `cal::local_date`:

```edgeql
SELECT <cal::local_date><json>'18870503';
```

This is fine because `<json>` turns it into a JSON string, and `cal::local_date` can be created from a string. The result we get is `{<cal::local_date>'1887-05-03'}`. But if we try to turn the JSON value into an `int64`, it won't work:

```edgeql
SELECT <int64><json>'18870503';
```

The problem is that it is a conversion from a JSON string to an EdgeDB `int64`. It gives this error: `ERROR: InvalidValueError: expected json number, null; got json string`. To keep things symmetrical, you need to cast a JSON string to an EdgeDB `str` and then cast into an `int64`:

```edgeql
SELECT <int64><str><json>'18870503';
```

Now it works: we get `{18870503}` which began as an EdgeDB `str`, turned into a JSON string, then back into an EdgeDB `str`, and finally was cast into an `int64`.

The [documentation on JSON](https://www.edgedb.com/docs/datamodel/scalars/json) explains which JSON types turn into which EdgeDB types and is good to bookmark if you need to convert from JSON a lot. You can also see a list of JSON functions [here](https://www.edgedb.com/docs/edgeql/funcops/json).

[Here is all our code so far up to Chapter 6.](code.md)

<!-- quiz-start -->

## Time to practice

1. This select is incomplete. How would you complete it so that it says "Pleased to meet you, I'm " and then the NPC's name?

   ```edgeql
   SELECT NPC {
     name,
     greeting := ## Put the rest here
   };
   ```

2. How would you update Mina's `places_visited` to include Romania if she went to Castle Dracula for a visit?

3. With the set `{'W', 'J', 'C'}`, how would you display all the `Person` types with a name that contains any of these capital letters?

   Hint: it involves `WITH` and a bit of concatenation.

4. How would you display this same query as JSON?

5. How would you add ' the Great' to every Person type?

   Bonus question: what's a quick way to undo this using string indexing?

[See the answers here.](answers.md)

<!-- quiz-end -->

__下一章：__ _乔纳森爬上城墙进入伯爵的房间。_
