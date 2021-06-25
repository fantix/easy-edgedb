# Chapter 16 Questions and Answers

#### 1. 如何将所有名称是由两个单词构成的 `Person` 对象的名称拆分成两个字符串，并忽略掉那些不完全是两个词的？

你可以使用函数 `str_split()` 用空格进行分割，然后根据结果的长度进行过滤：

```edgeql
WITH two_names := (SELECT str_split(Person.name, ' '))
SELECT two_names FILTER len(two_names) = 2;
```

#### 2. 如何显示所有名称里有“ma”的 `Person` 对象的名称？

这个查询很短：

```edgeql
SELECT (Person.name, find(Person.name, 'ma'));
```

请注意，第一个 `Person.name` 和第二个 `Person.name` 是相同的，这就是不使用笛卡尔乘法的原因。但是，如果你将第二个更改为 `DETACHED Person.name`，你将获得超过 100 个结果。

#### 3. 如何对 `Person` 类型的 `pen_name` 属性进行索引？

你可以直接通过使用 `index on (.pen_name)` 来实现。

也可以处于某种原因将它指定为下面这样的表达式：

```sdl
index on (.name ++ ' ' ++ .degrees IF EXISTS .degrees ELSE .name)
```

#### 4. 如何对每个 `Person` 的名称先以大写字母显示，再跟一个空格及其名称的小写形式？

这里给出可能是最简单的方法 —— 只需使用 `str_upper` 和 `str_lower` 方法来完成，并使用 `++' '` 再中间添加一个空格：

```edgeql
SELECT str_upper(Person.name) ++ ' ' ++ str_lower(Person.name);
```

#### 5. 如何使用 `re_match_all()` 来显示名称中带有 `Crewman` 的所有 `Person.name`？例如：Crewman 1，Crewman 2，等等。

你可以通过使用 `.` 通配符来匹配任何东西：

```edgeql
SELECT re_match_all('Crewman .', Person.name);
```
