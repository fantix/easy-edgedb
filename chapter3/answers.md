# Chapter 3 Questions and Answers

#### 1. 此查询试图显示每个 `NPC` 的 `name`，并给每个 `NPC` 加上所有 `City` 类型，但执行后给出了错误。请问它缺少了什么？

它所需要的只是 `SELECT` 语句前后的括号 —— 请记住，把它放在括号中来“捕获”输出，以便它可以被使用。

```edgeql
SELECT NPC {
  name,
  cities := (SELECT City.name)
};
```

#### 2. 如果 `City` 类型需要一个名为 `population` 的 `required property` 属性，它会是什么样子？“population”会是什么类型？

现在 `City` 只是扩展了 `Place`：

```sdl
type City extending Place;
```

因此它需要一个 `required property` 的“population”属性:

```sdl
type City extending Place {
  required property population -> int32
};
```

`int16` 最多只能达到 32,767，显然对于一个城市来说太少了。`int32` 和 `int64` 都足够大，且 `int32` 的最大值是 2,147,483,647，对于一个城市来说肯定是足够了。

#### 3. 此查询出于某种原因想要显示两次 `name`，但出现错误。你能想出办法吗？

你可以通过在第二次访问时给它一个不同的名称来访问 `property name` 两次。让我们叫它 name2：

```edgeql
SELECT Person {
  name,
  name2 := .name
};
```

#### 4. 有人一直在尝试给某个角色赋予负数年龄，你可以想出一种阻止这种情况的约束吗？

我们还没有见过这种约束，但很容易猜到：通过 `min_value()` 来约束。有了这两个约束，HumanAge 必须在 0 和 120 之间：

```sdl
scalar type HumanAge extending int16 {
  constraint max_value(120);
  constraint min_value(0);
}
```

#### 5. 你能插入一个 HumanAge 类型吗？

不能，因为它是一个标量类型，而不是一个对象 —— 一个 `HumanAge` 只是一个没有连接到任何的 `int16`。
No, because it's a scalar type and not an object - a `HumanAge` would just be an `int16` connected to nothing.

但是当然，你可以通过强制转换来选择一个：

```edgeql
SELECT <HumanAge>16;
```

这将给出 `{16}`。

通过尝试以下操作，你也可以看出，它仅仅是 `int16` 的不同名称。

```edgeql
SELECT <HumanAge>16 IS int16;
SELECT <HumanAge>16 IS HumanAge;
```

两句均会返回：`{true}`。
