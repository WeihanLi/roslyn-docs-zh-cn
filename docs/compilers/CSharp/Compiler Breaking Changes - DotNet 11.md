# .NET 10.0.100 至 .NET 11.0.100 之间 Roslyn 的重大更改

本文档列出了在 .NET 10 正式发布（.NET SDK 版本 10.0.100）之后到 .NET 11 正式发布（.NET SDK 版本 11.0.100）之间，Roslyn 中已知的重大更改。

## Span/ReadOnlySpan 类型集合表达式的*安全上下文*现在是*声明块*

***在 Visual Studio 2026 版本 18.3 中引入***

C# 编译器进行了重大更改，以正确遵守*集合表达式*功能规范中的 [ref 安全规则](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#ref-safety)。具体来说，是以下子句：

> * 如果目标类型是 *span 类型* `System.Span<T>` 或 `System.ReadOnlySpan<T>`，则集合表达式的安全上下文为*声明块*。

之前，编译器在这种情况下使用的是安全上下文*函数成员*。我们现在已更改为按规范使用*声明块*。这可能导致现有代码出现新错误，如以下场景：

```cs
scoped Span<int> items1 = default;
scoped Span<int> items2 = default;
foreach (var x in new[] { 1, 2 })
{
    Span<int> items = [x];
    if (x == 1)
        items1 = items; // 之前允许，现在报错

    if (x == 2)
        items2 = items; // 之前允许，现在报错
}
```

如果您的代码受到此重大更改的影响，请考虑改用数组类型作为相关集合表达式的类型：

```cs
scoped Span<int> items1 = default;
scoped Span<int> items2 = default;
foreach (var x in new[] { 1, 2 })
{
    int[] items = [x];
    if (x == 1)
        items1 = items; // 正常，使用 'int[]' 到 'Span<int>' 的转换

    if (x == 2)
        items2 = items; // 正常
}
```

或者，将集合表达式移到允许赋值的作用域：
```cs
scoped Span<int> items1 = default;
scoped Span<int> items2 = default;
Span<int> items = [0];
foreach (var x in new[] { 1, 2 })
{
    items[0] = x;
    if (x == 1)
        items1 = items; // 正常

    if (x == 2)
        items2 = items; // 正常
}
```

参见 https://github.com/dotnet/csharplang/issues/9750。

## 需要编译器合成 `ref readonly` 返回委托的场景现在需要 `System.Runtime.InteropServices.InAttribute` 类型可用

***在 Visual Studio 2026 版本 18.3 中引入***

C# 编译器进行了重大更改，以正确为返回 `ref readonly` 的[合成委托](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/lambda-improvements.md#delegate-types)发出元数据。

这可能导致现有代码中出现"error CS0518: 预定义类型 'System.Runtime.InteropServices.InAttribute' 未定义或未导入"，例如以下场景：

```cs
var d = this.MethodWithRefReadonlyReturn;
```

```cs
var d = ref readonly int () => ref x;
```

如果您的代码受到此重大更改的影响，请考虑向您的项目添加定义了 `System.Runtime.InteropServices.InAttribute` 的程序集的引用。

## 使用 `ref readonly` 局部函数的场景现在需要 `System.Runtime.InteropServices.InAttribute` 类型可用

***在 Visual Studio 2026 版本 18.3 中引入***

C# 编译器进行了重大更改，以正确为返回 `ref readonly` 的局部函数发出元数据。

这可能导致现有代码中出现"error CS0518: 预定义类型 'System.Runtime.InteropServices.InAttribute' 未定义或未导入"，例如以下场景：

```cs
void Method()
{
    ...
    ref readonly int local() => ref x;
    ...
}
```

如果您的代码受到此重大更改的影响，请考虑向您的项目添加定义了 `System.Runtime.InteropServices.InAttribute` 的程序集的引用。

## 不允许对左操作数静态类型为接口的 `&&`/`||` 运算符进行动态求值

***在 Visual Studio 2026 版本 18.3 中引入***

C# 编译器现在在接口类型用作逻辑 `&&` 或 `||` 运算符的左操作数且右操作数为 `dynamic` 时报告错误。
之前，对于具有 `true`/`false` 运算符的接口类型，代码可以编译，但由于运行时绑定器无法调用接口上定义的运算符，会在运行时抛出 `RuntimeBinderException`。

此更改通过在编译时报告错误来防止运行时错误。错误消息为：

> error CS7083: 表达式必须可隐式转换为 Boolean，或其类型 'I1' 不得为接口且必须定义运算符 'false'。

```cs
interface I1
{
    static bool operator true(I1 x) => false;
    static bool operator false(I1 x) => false;
}

class C1 : I1
{
    public static C1 operator &(C1 x, C1 y) => x;
    public static bool operator true(C1 x) => false;
    public static bool operator false(C1 x) => false;
}

void M()
{
    I1 x = new C1();
    dynamic y = new C1();
    _ = x && y; // error CS7083: 表达式必须可隐式转换为 Boolean，或其类型 'I1' 不得为接口且必须定义运算符 'false'。
}
```

如果您的代码受到此重大更改的影响，请考虑将左操作数的静态类型从接口类型改为具体类类型，或改为 `dynamic` 类型：

```cs
void M()
{
    I1 x = new C1();
    dynamic y = new C1();
    _ = (C1)x && y; // 有效 - 使用 C1 上定义的运算符
    _ = (dynamic)x && y; // 有效 - 使用 C1 上定义的运算符
}
```

参见 https://github.com/dotnet/roslyn/issues/80954。

## `nameof(this.)` 在属性中被禁止

***在 Visual Studio 2026 版本 18.3 和 .NET 10.0.200 中引入***

自 C# 12 起，在属性的 `nameof` 中使用 `this` 或 `base` 关键字在 Roslyn 中被无意地允许，
现在[已正确禁止](https://github.com/dotnet/roslyn/pull/81628)以符合语言规范。
可以通过删除 `this.` 并直接访问成员（不使用限定符）来缓解此重大更改。

参见 https://github.com/dotnet/roslyn/issues/82251。

```cs
class C
{
    string P;
    [System.Obsolete(nameof(this.P))] // 现在被禁止
    [System.Obsolete(nameof(P))] // 解决方法
    void M() { }
}
```

## switch 表达式分支中 'with' 的解析

***在 Visual Studio 2026 版本 18.4 中引入***

参见 https://github.com/dotnet/roslyn/issues/81837 和 https://github.com/dotnet/roslyn/pull/81863

之前，当编译器看到以下代码时，会将 `(X.Y)when` 视为转型表达式，即将上下文标识符 `when` 转型为 `(X.Y)`：

```c#
x switch
{
    (X.Y) when
}
```

这是不希望的，意味着简单的 `when` 检查模式（如 `(X.Y) when a > b =>`）无法正确解析。现在，这被视为常量模式 `(X.Y)` 后跟一个 `when` 子句。

## 集合表达式中的 `with()` 被视为集合构造*参数*

***在 Visual Studio 2026 版本 18.4 中引入***

当 `with(...)` 用作集合表达式中的元素，且 LangVersion 设置为 15 或更高时，它将被绑定为传递给用于创建集合的构造函数或工厂方法的参数，而不是对名为 `with` 的方法的调用。

要绑定到名为 `with` 的方法，请改用 `@with`。

```cs
object x, y, z = ...;
object[] items;

items = [with(x, y), z];  // C# 14：调用 with() 方法；C# 15：object[] 不支持参数，报错
items = [@with(x, y), z]; // 调用 with() 方法
object with(object a, object b) { ... }
```

## 指针类型不再需要 unsafe 上下文

***在 Visual Studio 2026 版本 18.7 中引入***

在未来的 C# 版本中（目前处于 `langversion:preview`），指针类型（例如 `int*`、`delegate*<void>`）不再需要 unsafe 上下文。
只有指针间接操作（解引用、通过 `->` 的成员访问、元素访问等）需要 unsafe。
这是 [unsafe 演进](https://github.com/dotnet/csharplang/issues/9704)功能的一部分。

由于指针类型现在在安全上下文中合法，重载解析可能会考虑以前被排除的候选项。这可能导致新的歧义错误：

```cs
using System;

class Program
{
    static void Main()
    {
        M(x => { }); // C# 14：输出 "2"；C# preview：error CS0121（歧义）
    }

    static void M(F1 f) { Console.WriteLine(1); }
    static void M(F2 f) { Console.WriteLine(2); }
}

unsafe delegate void F1(int* x);
delegate void F2(int x);
```

之前，lambda `x => { }` 无法在安全上下文中转换为 `F1`（因为 `int*` 需要 unsafe 上下文），因此只有 `M(F2)` 适用。
现在 `int*` 在安全上下文中合法，lambda 可以转换为两个委托，产生歧义错误。

如果您的代码受到歧义更改的影响，请在 lambda 中添加显式参数类型以消除歧义：

```cs
M((int x) => { }); // 解析为 M(F2)
```

## `SkipLocalsInit` 中未初始化的 `stackalloc` 需要 unsafe 上下文

***在 Visual Studio 2026 版本 18.7 中引入***

在未来的 C# 版本中（目前处于 `langversion:preview`），在标有 `[SkipLocalsInit]` 的方法内没有初始化器的 `stackalloc` 表达式现在需要 unsafe 上下文，即使目标类型是 `Span<T>`。
这是因为 `SkipLocalsInit` 阻止了分配内存的零初始化，从而可以读取未初始化的数据——这是内存安全问题。
这是 [unsafe 演进](https://github.com/dotnet/csharplang/issues/9704)功能的一部分。

```cs
[System.Runtime.CompilerServices.SkipLocalsInit]
void M()
{
    Span<int> a = stackalloc int[5];           // 之前正常，现在 error CS9361
    Span<int> b = stackalloc int[] { 1, 2 };   // 正常（有初始化器）
    Span<int> c = stackalloc int[2] { 1, 2 };  // 正常（有初始化器）
}
```

如果您的代码受到影响，您可以：
- 在 `stackalloc` 周围添加 `unsafe` 块：
  ```cs
  [SkipLocalsInit]
  void M()
  {
      Span<int> a;
      unsafe { a = stackalloc int[5]; }
  }
  ```
- 或提供初始化器以确保内存完全初始化：
  ```cs
  [SkipLocalsInit]
  void M()
  {
      Span<int> a = stackalloc int[5] { 0, 0, 0, 0, 0 };
  }
  ```
