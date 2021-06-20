# Chapter 12 Questions and Answers

#### 1. 考虑下面这两个函数。EdgeDB 会接受第二个吗？

不会接受，因为它们的输入签名是相同的：

```sdl
function gives_number(input: int64) -> int64
  using(input);

function gives_number(input: int64) -> int32
  using(<int32>input);
```

因为，如果两者都可以使用 `int64`，则在输入后，EdgeDB 将无法知道你要使用两者中的哪一个。

这里是错误信息：

```
error: cannot create the `default::gives_number(input: std::int64)` function: a function with the same signature is already defined
```

#### 2. 那么下面两个函数呢？EdgeDB 会接受第二个吗？

会接受，它们有不同的签名，因为输入不同：一个采用 `int16`，另一个采用 `int32`。事实上，您可以通过使用 `DESCRIBE` 来查看它们。例如，`DESCRIBE FUNCTION make64 AS TEXT` 则会给出以下内容：

```
{
  'function default::make64(input: std::int16) ->  std::int64 using (input);
   function default::make64(input: std::int32) ->  std::int64 using (input);',
}
```

但是请注意，`SELECT make64(8);` 实际上会产生错误！错误是：

```
error: could not find a function variant make64
```

那是因为 `SELECT make64(8)` 输入的是一个 `int64`（默认），且它没有签名。你需要使用 `SELECT make64(<int32>8);`（或 `<int16>`）进行转换以使其工作。

#### 3. `SELECT {} ?? {3, 4} ?? {5, 6};` 能工作吗？

不能工作，因为 EdgeDB 不知道 `{}` 是什么类型。但是，如果你将 `{}` 转换为 `int64`：

```edgeql
SELECT <int64>{} ?? {3, 4} ?? {5, 6};
```

它会给出输出：`{3, 4}`。

#### 4. `SELECT <int64>{} ?? <int64>{} ?? {1, 2};` 能工作吗

可以工作，输出是 `{1, 2}`。如果你使用更多 `??`，它会一直运行，直到碰到不为空的东西。

知道了这一点，你大概可以猜到下面这个输出结果：

```edgeql
SELECT <int64>{} ?? <int64>{} ?? {1} ?? <int64>{} ?? {5};
```

输出是 `{1}`，它是第一个遇见的非空集。

#### 5. `SELECT array_join(array_agg(Person.name));` 在尝试获得一个含有所有人姓名的字符串，但它不能工作，问题出在哪里？

错误信息给出了一个提示：

`error: could not find a function variant array_join`

这意味着它收到的输入无法匹配到其任何函数签名。如果你查看其 [函数签名](https://www.edgedb.com/docs/edgeql/funcops/array#function::std::array_join)，你就会明白为什么了：它需要第二个字符串：

```sdl
std::array_join(array: array<str>, delimiter: str) -> str
```

因此，我们可以将其更改为 `SELECT array_join(array_agg(Person.name), ' ');` 或 `SELECT array_join(array_agg(Person.name), ' is awesome ');` 或其他任何字符串作为第二个字符串输入，它都将工作。
