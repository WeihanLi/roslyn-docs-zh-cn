冲突的枚举成员
=====================

如 [#2909](https://github.com/dotnet/roslyn/issues/2909) 中报告的，可以从 VB 引用包含多个同名枚举成员的枚举类型。例如，库作者可能在 C# 中发布其 API 的版本 1，包含此类型

```cs
enum Something
{
    Something,
    Datetime,
    SomethingElse,
}
```

然后后来意识到名称 Datetime 拼写错误（应该是 PascalCase，`Time` 大写）。所以在版本 2 中类型更改为

```cs
enum SomeEnum
{
    Something,
    DateTime,
    [Obsolete] Datetime = DateTime,
    SomethingElse,
}
```

使用旧枚举成员名称的 C# 客户端将收到编译器的过时警告，鼓励使用新名称而不是旧名称，但否则能够编译和运行未更改的代码。

使用 `SomeEnum.Datetime` 的 VB 客户端从以前版本的 VB 编译器不会收到任何警告。

```vb
    Dim v = SomeEnum.Datetime
```

虽然这个程序在技术上是错误的（名称 `Datetime` 在两个枚举成员之间是歧义的），Dev12 VB 编译器在枚举类型内查找枚举常量时会选择给定（不区分大小写）名称的*词法上第一个*枚举常量。由于该成员在此示例中没有 Obsolete 属性，VB 代码将继续编译和运行而没有问题。

Roslyn 编译器重现了这个 bug - 但仅当冲突的枚举常量具有相同的基础值时，如上面的示例 - 以便利用此功能的现有代码不会被破坏。例如，参见
- https://bugs.mysql.com/bug.php?id=37406
- http://lists.mysql.com/commits/48012
