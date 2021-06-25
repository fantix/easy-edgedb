---
tags: Indexing, String Functions
---

# 第十六章 - 雷菲尔德说的是实话吗？

> 亚瑟·霍姆伍德（Arthur Holmwood）的父亲去世了，现在亚瑟是一家之主。他的新头衔是戈达尔明勋爵（Lord Godalming），他有很多钱。用这笔钱，他可以帮助大家到出德古拉（Dracula）藏匿那些箱子的房子。

> 与此同时，范海辛（Van Helsing）很好奇，问约翰·西沃德（John Seward）是否可以见见伦菲尔德（Renfield）。他很惊讶地看到伦菲尔德受过很好的教育，而且口才很好。伦菲尔德谈论着范海辛的研究、政治、历史等 —— 他看起来一点都不疯狂！但后来，伦菲尔德又不想说话了，只是骂他是个白痴。令人很困惑。一天晚上，伦菲尔德非常认真地要求他们让他离开。他说：“你们难道不知道我是理智而认真的吗？……一个为灵魂而战的正常人？听我的，听我的，让我走！让我走！”。他们想相信他，但又无法相信他。最后，伦菲尔德停了下来并平静地说：“记住，我今晚已经尽力说服你们了。” 

## `index on` 用于更快的查询（for quicker lookups）

我们快到书的结尾了，但还有很多数据尚未输入。书中还有很多可能有用的数据，但我们还没有准备好进行整理。幸运的是，原版小说《德古拉》是以书信、日记等形式组织的，均以日期或时间开头。它们都是以下面这种方式开始的：

```
Dr. Seward’s Diary.
1 October, 4 a. m.—Just as we were about to leave the house...

Letter, Van Helsing to Mrs. Harker.
“24 September.
“Dear Madam...

Mina Murray’s Journal.
8 August. — Lucy was very restless all night, and I, too, could not sleep...
```

这对我们来说非常方便。有了这个，我们可以创建一个类型来保存日期和书中的字符串，供我们以后搜索。我们称之为 `BookExcerpt`（excerpt = 摘录 = 一本书的一部分）。

```sdl
type BookExcerpt {
  required property date -> cal::local_datetime;
  required property excerpt -> str;
  index on (.date);
  required link author -> Person
}
```

[`index on (.date)`](https://www.edgedb.com/docs/datamodel/indexes#indexes) 部分是之前没见过的，它意味着创建一个索引以加快未来的查询速度。使用 `index on` 查找更快，是因为现在数据库不需要按顺序扫描整个对象集来找到匹配的对象。与总是扫描所有内容相比，索引使精确匹配查找速度更快。

我们也可以对某些其他类型执行此操作 —— 它可能适用于诸如 `Place` 和 `Person` 之类的类型。

注意：`index` 在数量有限的情况下是表现良好的，且你不会希望索引所有的东西，因为：

- 它使查询更快了，但增加了数据库大小。
- 如果你有太多，这可能会使 `INSERT` 和 `UPDATE` 变慢。

这可能并不奇怪，因为你可以看到 `index` 是用户需要做出的选择。如果在每个情况下使用 `index` 都是最好的主意，那么 EdgeDB 会自动执行。

最后，这里有两个不需要创建 `index` 的情况：

- 在链接上（on links），
- 在有排他性约束的属性上。

在这两种情况下索引都会被自动创建，因此你无需为它们使用索引。

那么，让我们来插入两条摘录。这些条目中的字符串很长（有时会长达几页），因此我们将在此处仅显示开头和结尾：

```edgeql
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 4, 0, 0),
  author := (SELECT Person FILTER .name = 'John Seward' LIMIT 1),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};
```

```edgeql
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 5, 0, 0),
  author := (SELECT Person FILTER .name = 'Jonathan Harker' LIMIT 1),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```

稍后我们可以执行查询，按时间顺序获取所有条目并显示为 JSON。

```edgeql
SELECT <json>(
  SELECT BookExcerpt {
    date,
    author: {
      name
    },
    excerpt
  } ORDER BY .date
);
```

这是仅包含一小部分摘录的 JSON 输出：

```
{
  "{\"date\": \"1887-10-01T04:00:00\", \"author\": {\"name\": \"John Seward\"}, \"excerpt\": \"Dr. Seward\'s Diary.\\n 1 October, 4 a.m... -- Just as we were about to leave the house...\\\"\"}",
  "{\"date\": \"1887-10-01T05:00:00\", \"author\": {\"name\": \"Jonathan Harker\"}, \"excerpt\": \"1 October, 5 a.m. -- I went with the party to the search...\"}",
}
```

然后，我们可以添加一个链接到我们的 `Event` 类型，并连到我们新的类型 —— `BookExcerpt`。于是，`Event` 现在看起来是这样：

```sdl
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  multi link excerpt -> BookExcerpt; # Only this is new
  property exact_location -> tuple<float64, float64>;
  property east_west -> bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ 'E' if .east = true else 'W';
}
```

你可以看到 `description` 是我们写的一个短字符串，而 `excerpt` 则链接到直接来自原著中的较长文本。

## 更多字符串函数（More functions for strings）

[字符串函数](https://www.edgedb.com/docs/edgeql/funcops/string) 在对我们的 `BookExcerpt` 类型（或通过 `Event` 的 `BookExcerpt`）进行查询时特别有用。一个是函数 [`str_lower()`](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::str_lower) ，它可以使字符串都变为小写：

```edgeql-repl
edgedb> SELECT str_lower('RENFIELD WAS HERE');
{'renfield was here'}
```

下面是一个较长的查询：

```edgeql
SELECT BookExcerpt {
  excerpt,
  length := (<str>(SELECT len(.excerpt)) ++ ' characters'),
  the_date := (SELECT (<str>.date)[0:10]),
} FILTER contains(str_lower(.excerpt), 'mina');
```

- 对 `.excerpt` 使用了 `len()`，然后将其转换为字符串；
- 使用 `str_lower()` 将 `.excerpt` 转换为全部小写，并与 `mina` 进行比较；
- 将 `cal::local_datetime` 切片成一个字符串，使其只打印索引 0 到 10。

这是输出结果：

```
{
  Object {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1887-10-01',
  },
}
```

制作 `the_date` 的另一种方法是使用 [to_str](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::to_str) 函数，它（你可能猜到了）会将其变成一个字符串：

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(SELECT len(.excerpt)) ++ ' characters'),
  the_date := (SELECT to_str(.date)), #only this part is different
} FILTER contains(str_lower(.excerpt), 'mina');
```

字符串的其他一些函数有：

- `find()`：给出找到的第一个匹配项的索引，如果找不到任何匹配内容，则返回 `-1`：

`SELECT find(BookExcerpt.excerpt, 'sofa');` 输出 `{-1, 151}`。这是因为第一个 `BookExcerpt.excerpt` 中没有单词 `sofa`，而第二个里在索引 151 处有 `sofa`。

- `str_split()`：可以将字符串以你想要的方式进行拆分并转变为数组。最常见的是用 `' '` 分割来分隔单词：

```edgeql-repl
edgedb> SELECT str_split('Oh, hear me! hear me! Let me go! let me go! let me go!', ' ');

{
  [
    'Oh,',
    'hear',
    'me!',
    'hear',
    'me!',
    'Let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
  ],
}
```

下面的例子也同样工作（以 `'n'` 进行分割）：

```edgeql
SELECT MinorVampire {
  names := (SELECT str_split(.name, 'n'))
};
```

现在 `n` 都不见了：

```
{
  default::MinorVampire {names: ['Woma', ' 1']},
  default::MinorVampire {names: ['Woma', ' 2']},
  default::MinorVampire {names: ['Woma', ' 3']},
  default::MinorVampire {names: ['Lucy Weste', 'ra']},
}
```

你也可以以 `\n` 为分割按新行进行拆分。虽然你看不到它（`\n`），但从计算机的角度来看，每条新行中都有一个 `\n`。因此：

```edgeql
SELECT str_split('Oh, hear me!
hear me!
Let me go!
let me go!
let me go!', '\n');
```

will split it by line and give the following array:

```
{['Oh, hear me! ', 'hear me! ', 'Let me go! ', 'let me go! ', 'let me go!']}
```

- `re_match()`（用于第一次匹配）和 `re_match_all()`（用于所有匹配）：如果你了解如何使用 [正则表达式](https://en.wikipedia.org/wiki/Regular_expression)，并想使用它们。这俩函数可能很有用，因为这本小说写于 100 多年前，有些拼写已经不大一样了。例如，`tonight` 这个词在《德古拉》中总是使用较旧的 `to-night` 拼写。为了处理这样的问题，我们就可以使用这两个函数：

```edgeql-repl
edgedb> SELECT re_match_all('[Tt]o-?night', 'Dracula is an old book, so the word tonight is written to-night. Tonight we know how to write both tonight and to-night.');
{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}
```

这个函数签名是`std::re_match(pattern: str, string: str) -> array<str>`，正如你所看到的那样，第一个参数是模式（pattern），然后是一个字符串。模式 `[Tt]o-?night` 的意思是：

- 以 `T` 或 `t` 开头，
- 然后有一个 `o`，
- 也许中间有一个 `-`，
- 并以 `night` 结束，

所以它给出：`{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}`。

如果想匹配任何内容，你可以使用通配符：`.`。

## 更多关于 `index on`（Two more notes on `index on`）

顺便说一下，`index on` 也可以用在你自己制作的表达式上。在我们知道所有这些字符串函数后，这尤其有用。例如，如果我们总是需要查询一个 `City` 的名称及其人口，我们可以通过下面这种方式进行索引：


```sdl
type City extending Place {
  annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
  property population -> int64;
  index on (.name ++ ': ' ++ <str>.population);
}
```

另外不要忘记，你也可以为此添加注释。如果你认为代码的读者可能不知道它的用途，`(.name ++ ': ' + <str>.population)` 则可能是一个很好的注释案例：

```
type City extending Place {
    annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
    property population -> int64;
    index on (.name ++ ': ' ++ <str>.population) {
      annotation title := 'Lists city name and population for use in game function get_city_names';
    }
}
```

其中 `get_city_names` 不是一个真正的函数；我们只是假装它在游戏中的某个地方使用过并且重要到应该记住。

[→ 点击这里查看第 16 章相关代码](code.md)

<!-- quiz-start -->

## 小测验

1. 如何将所有名称是由两个单词构成的 `Person` 对象的名称拆分成两个字符串，并忽略掉那些不完全是两个词的？

2. 如何显示所有名称里有“ma”的 `Person` 对象的名称？

   提示：使用函数 `find()`。

3. 如何对 `Person` 类型的 `pen_name` 属性进行索引？

   提示：尝试使用 `describe type Person as SDL` 查看一下它的 `pen_name` 属性。

4. 如何对每个 `Person` 的名称先以大写字母显示，再跟一个空格及其名称的小写形式？

   提示：[str_repeat()](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::str_repeat) 函数可以提供帮助（尽管还有很多其他方法）

5. 如何使用 `re_match_all()` 来显示名称中带有 `Crewman` 的所有 `Person.name`？例如：Crewman 1，Crewman 2，等等。

   提示：如果你想快速了解正则表达式，[这里有一些基本概念](https://en.wikipedia.org/w/index.php?title=Regular_expression&oldid=988356211#Basic_concepts) 供你阅读。

[点击这里查看答案](answers.md)

<!-- quiz-end -->

__接下来：__ _关于伦菲尔德的真相。_
