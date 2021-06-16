# Chapter 8 Questions and Answers

#### 1. 如何选择出所有 `Place` 和他们的名字，以及当它是个 `Castle` 时的属性 `door`？

答案如下：

```edgeql
SELECT Place {
  name,
  [IS Castle].doors
};
```

没有 `[IS Castle]`，将无法正常执行。

#### 2. 如何选择出 `Place`，用 `city_name` 显示当它是个 `City` 时的 `name`，并用 `country_name` 显示当它是个 `Country` 时的 `country_name`？

```edgeql
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
};
```

#### 3. 基于上一题，如何做可以只显示属于 `City` 或 `Country` 类型的结果？

这个问题是基于问题 2 给出的这个结果：

```
{
  Object {city_name: {}, country_name: {}},
  Object {city_name: 'Munich', country_name: {}},
  Object {city_name: 'Buda-Pesth', country_name: {}},
  Object {city_name: 'Bistritz', country_name: {}},
  Object {city_name: 'London', country_name: {}},
  Object {city_name: {}, country_name: 'Romania'},
  Object {city_name: {}, country_name: 'Slovakia'},
  Object {city_name: {}, country_name: {}},
}
```

像 `Object {city_name: {}, country_name: {}},` 这样的结果对我们没有什么用，我们可以用 `EXISTS` 将它们过滤掉： 

```edgeql
SELECT Place {
  city_name := [IS City].name,
  country_name := [IS Country].name
} FILTER EXISTS .city_name OR EXISTS .country_name;
```

另一种过滤方式是使用 `FILTER Place IS City | Country`。你可能熟悉其他编程语言中的 `|`。在 EdgeDB 中，这称为类型联合运算符（the type union operator），你将在第 13 章中了解有关它的更多信息。

#### 4. 你将如何显示所有没有 `lover` 的 `Person` 对象及其名称和类型名称？

要获得所有这些单身人士、姓名和对象类型，只需执行以下操作：

```edgeql
SELECT Person {
  name,
  __type__: {
    name
  }
} FILTER NOT EXISTS .lover;
```

Don't forget `name` after type! It won't make an error but the type name will be something like this and not very helpful: `__type__: Object {id: 20ef52ae-1d97-11eb-8cb6-0de731b01cc9}`

#### 5. 下面这个查询需要修复什么？提示：有两个地方是必须要修复的，还有一个地方可能应该更改以使其更具可读性。

需要修复的两个部分是：1) `name` 后面加上 `,`，2) `[IS Castle]` 后面加上 `.`：

```edgeql
SELECT Place {
  __type__,
  name,
  [IS Castle].doors
};
```

_应该_ 修复的部分是：我们可能应该将 `name` 放在 `__type__` 中，以便我们可以读懂它（而不是 `__type__: Object {id: e0a9ab38-1e6e-11eb-9497-5bb5357741af}`）。 如下所示：

```edgeql
SELECT Place {
  __type__: {
    name
  },
  name,
  [IS Castle].doors
};
```
