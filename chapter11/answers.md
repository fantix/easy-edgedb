# Chapter 11 Questions and Answers

#### 1. 如何编写一个名为 `lucy()` 的函数，它将只返回与名称“Lucy Westenra”匹配的所有 `NPC` 类型？

这很简单，只是别忘记返回 `NPC`（或者 `Person`，取决于你的喜好）：

```sdl
function lucy() -> NPC
  using (
    SELECT NPC FILTER .name = 'Lucy Westenra'
  );
```

然后你可以按照如下方式使用它：

```edgeql
SELECT lucy() {
  name,
  places_visited: {name}
};
```

#### 2. 如何编写一个函数，其接受两个字符串并会返回名称与输入的两个字符串任意一个匹配的所有 Person 对象？

让我们将函数命名为 `get_two()`。定义如下所示：

```sdl
function get_two(one: str, two: str) -> SET OF Person
  using (
    WITH person_1 := (SELECT Person filter .name = one LIMIT 1),
         person_2 := (SELECT Person filter .name = two LIMIT 1),
    SELECT {person_1, person_2}
  );
```

下面，该函数将用于对 John Seward、Count Dracula 和他们的奴隶姓名的查询：

```edgeql
SELECT get_two('John Seward', 'Count Dracula') {
  name,
  [IS Vampire].slaves: {name},
};
```

这是输出结果：

```
{
  Object {name: 'John Seward', slaves: {}},
  Object {
    name: 'Count Dracula',
    slaves: {
      Object {name: 'Woman 1'},
      Object {name: 'Woman 2'},
      Object {name: 'Woman 3'},
    },
  },
}
```

#### 3. 下面的语句将会输出什么？

查看输入后，你可以看到有 16 行信息生成，因为每个集合的每个部分都会与其他每个集合的每个部分进行组合：

```edgeql
SELECT {'Jonathan', 'Arthur'} ++ {' loves '} ++ {'Mina', 'Lucy'} ++ {' but '} ++ {'Dracula', 'The inkeeper'} ++ {' doesn\'t love '} ++ {'Mina', 'Jonathan'};
```

(2 * 1 * 2 * 1 * 2 * 1 * 2 = 16)

下面是输出的结果：

```
{
  'Jonathan loves Mina but Dracula doesn\'t love Mina',
  'Jonathan loves Mina but Dracula doesn\'t love Jonathan',
  'Jonathan loves Mina but The inkeeper doesn\'t love Mina',
  'Jonathan loves Mina but The inkeeper doesn\'t love Jonathan',
  'Jonathan loves Lucy but Dracula doesn\'t love Mina',
  'Jonathan loves Lucy but Dracula doesn\'t love Jonathan',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Mina',
  'Jonathan loves Lucy but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Mina but Dracula doesn\'t love Mina',
  'Arthur loves Mina but Dracula doesn\'t love Jonathan',
  'Arthur loves Mina but The inkeeper doesn\'t love Mina',
  'Arthur loves Mina but The inkeeper doesn\'t love Jonathan',
  'Arthur loves Lucy but Dracula doesn\'t love Mina',
  'Arthur loves Lucy but Dracula doesn\'t love Jonathan',
  'Arthur loves Lucy but The inkeeper doesn\'t love Mina',
  'Arthur loves Lucy but The inkeeper doesn\'t love Jonathan',
}
```

#### 4. 如何制作一个函数来计算一个城市比另一个城市大多少倍？

这是一种方法：

```sdl
function two_cities(city_one: str, city_two: str) -> float64
  using (
    WITH first_city := (SELECT City filter .name = city_one),
         second_city := (SELECT City filter .name = city_two),
    SELECT first_city.population / second_city.population
  );
```

然后它将如下所示般被使用：

```edgeql-repl
edgedb> SELECT two_cities('Munich', 'Bistritz');
{25.277252747252746}
edgedb> SELECT two_cities('Munich', 'London');
{0.06572085714285714}
```

`city_one` 和 `city_two` 当然也可以定义为 `City` 类型，但是使用起来可能会有些别手。

#### 5. `SELECT (City.population + City.population)` 和 `SELECT ((SELECT City.population) + (SELECT City.population))` 会产生不同的结果吗？

> 第十章中我们定义过五个城市的人口，在这里引用，便于回忆：
> * Buda-Pesth (Budapest): 402706
> * London: 3500000
> * Munich: 230023
> * Whitby: 14400
> * Bistritz (Bistrița): 9100

会的。第一个是单个集合上的 `SELECT`，会将每个项目添加到自身，结果是：`{28800, 460046, 805412, 18200, 7000000}`

同时，第二个是作用在两个集合上，其中每个项目都会分别和另一个集合里的每个项目进行一次相加。因此会得到 25（5 * 5）个结果：

```
{
  28800,
  244423,
  417106,
  23500,
  3514400,
  244423,
  460046,
  632729,
  239123,
  3730023,
  417106,
  632729,
  805412,
  411806,
  3902706,
  23500,
  239123,
  411806,
  18200,
  3509100,
  3514400,
  3730023,
  3902706,
  3509100,
  7000000,
}
```
