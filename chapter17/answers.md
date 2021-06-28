# Chapter 17 Questions and Answers

#### 1. 如何显示每个 NPC 的名称、力量值、所访问城市的名称和人口，以及年龄（如果年龄 = `{}`，则显示 0）？试试只用单行代码。

这可以在一行上完成，但别忘了，我们需要一个 `[IS City]`，因为 `places_visited` 链接到 `Place`，但只有 `City` 有人口属性。如下所示：

```edgeql
SELECT (
  NPC.name,
  NPC.strength,
  (NPC.places_visited[IS City].name, NPC.places_visited[IS City].population),
  NPC.age IF EXISTS NPC.age ELSE 0
);
```

#### 2. 上题的查询显示了很多没有任何上下文的数字。我们应该做些什么使其更友好？

上题的查询的执行结果是：

```
{
  ('Mina Murray', 0, ('London', 3500000), 0),
  ('John Seward', 0, ('London', 3500000), 0),
  ('Quincey Morris', 0, ('London', 3500000), 0),
  # ...
}
```

我们该做的当然是“命名元组”：

```edgeql
SELECT (
  name := NPC.name,
  strength := NPC.strength,
  city_populations := (NPC.places_visited[IS City].name, NPC.places_visited[IS City].population),
  age := NPC.age if EXISTS NPC.age else 0
);
```

现在输出看起来好多了：

```
{
  (name := 'Mina Murray', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'John Seward', strength := 0, city_populations := ('London', 3500000), age := 0),
  (name := 'Quincey Morris', strength := 0, city_populations := ('London', 3500000), age := 0),
}
```

如果需要，你甚至可以使用城市名称和人口命名元组内的元组。

#### 3. 伦菲尔德（Renfield）现在已经死了，需要一个 `last_appearance`。尝试编写一个名为 `make_dead(person_name: str, date: str) -> Person` 的函数，你只需要输入角色的名字和日期就即可。

你可以这样编写：

```sdl
function make_dead(person_name: str, date: str) ->  Person {
  using (
    WITH dead_person := (
        SELECT default::Person
        FILTER (.name = person_name)
        LIMIT 1
      )
    UPDATE dead_person
    SET {
      last_appearance := <cal::local_date>date
    }
  );
```
