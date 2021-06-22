# Chapter 13 Questions and Answers

#### 1. 尝试用一个单独的插入语句插入一个名为“Mr. Swales”的 `NPC`，他曾到访过名为“York”的 `City`，名为“England”的  `Country` 以及名为“Whitby Abbey”的 `OtherPlace`。

类似于我们之前章节中所做的 `Ship` 插入：

```edgeql
INSERT NPC {
  name := 'Mr. Swales',
  places_visited := {
    (INSERT City {
      name := 'York'
    }),
    (INSERT Country {
      name := 'England'
    }),
    (INSERT OtherPlace {
      name := 'Whitby Abbey'
    }),
  }
};
```

#### 2. 这个内省查询的可读性如何？

这个查询：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links
};
```

三分之一可读：`name` 实际上会显示为人类可读的名称。结果如下：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 70719b55-2031-11eb-b0e6-81bc5b4ee64b},
      schema::Property {id: 7075a94c-2031-11eb-9cb0-6d25e19e0e31},
    },
    links: {
      schema::Link {id: 70798bba-2031-11eb-b091-1d60d8428852},
      schema::Link {id: 70740220-2031-11eb-9bd0-f94fde12382f},
      schema::Link {id: 7076b0c7-2031-11eb-9cab-b7b889902e39},
    },
  },
}
```

在两个地方添加 `: {name}` 则可使其完全可读：

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {name},
  links: {name},
};
```

结果是：

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id'},
      schema::Property {name: 'name'},
    },
    links: {
      schema::Link {name: 'crew'},
      schema::Link {name: '__type__'},
      schema::Link {name: 'sailors'},
    },
  },
}
```

#### 3. 查看 `Vampire` 类型有哪些链接的最简短的方法是什么？

类似于 `SELECT Vampire.name` 给出 `Vampire` 类型的所有名称（与 `SELECT Vampire { name }` 相反），你可以这样做：

```edgeql
SELECT (Introspect Vampire).links { name };
```

输出是：

```
{
  schema::Link {name: 'slaves'},
  schema::Link {name: '__type__'},
  schema::Link {name: 'lover'},
  schema::Link {name: 'places_visited'},
}
```

#### 4. 你认为 `SELECT DISTINCT {1, 2} + {1, 2};` 的输出会是什么？

输出是：

```
{2, 3, 3, 4}
```

你可以看到 `DISTINCT` 是独立作用于一个集合的（所以这里只是作用在了第一个上），所以 `SELECT DISTINCT {1, 2} + {1, 2};` 和 `SELECT {1, 2} + {1, 2};` 是相同的。如果你要写 `SELECT DISTINCT {2, 2}`，输出将只是 `{2}`。

#### 5. 你认为 `SELECT DISTINCT {2, 2} + {2, 2};` 的输出会是什么？

输出将为 `{4, 4}`，因为 `DISTINCT` 仅适用于第一个集合。

要获得输出 `{4}`，你可以重复 `DISTINCT`：`DISTINCT`: `SELECT DISTINCT {2, 2} + DISTINCT {2, 2};`。或者你可以像这样包装整个运算：`SELECT DISTINCT({2, 2} + {2,2})`。
