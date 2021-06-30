# Chapter 19 Questions and Answers

#### 1. 如何显示所有 `City` 名称和它们所在的 `Region` 名称？

现在这对我们来说不是太难，只是一个反向查找。我们还将添加一个过滤器，只显示位于存在的 `Region` 中的 `City` 对象：

```edgeql
SELECT City {
  name,
  region_name := .<cities[IS Region].name
} FILTER EXISTS .region_name;
```

#### 2. 基于上一题，如何显示所有 `City` 名称和它们所在的 `Region` 名称及 `Country` 名称？

这是一个类似的查询，只是这次我们需要返回两个链接。

`.<cities[IS Region].name` 的意思是“通过一个名为 `cities` 的链接连接的 `Region` 的名称”，对 `Country` 使用同样的方式。如下所示：

`.<cities[IS Region].<regions[IS Country].name`

换句话说，是指“通过一个名为 `cities` 的链接链接到的 `Region`，这些 `Region` 再通过一个名为 `regions` 的链接连接到的 `Country` 的名称"。

```edgeql
SELECT City {
  name,
  region_name := .<cities[IS Region].name,
  country_name := .<cities[IS Region].<regions[IS Country].name
} FILTER EXISTS .country_name;
```

这为我们提供了不错的、从 `City` 到 `Country` 级别的输出：

```
{
  Object {name: 'Dresden', region_name: {'Saxony'}, country_name: {'Germany'}},
  Object {name: 'Leipzig', region_name: {'Saxony'}, country_name: {'Germany'}},
  Object {name: 'Darmstadt', region_name: {'Hesse'}, country_name: {'Germany'}},
  Object {name: 'Mainz', region_name: {'Hesse'}, country_name: {'Germany'}},
  Object {name: 'Berlin', region_name: {'Prussia'}, country_name: {'Germany'}},
  Object {name: 'Königsberg', region_name: {'Prussia'}, country_name: {'Germany'}},
}
```
