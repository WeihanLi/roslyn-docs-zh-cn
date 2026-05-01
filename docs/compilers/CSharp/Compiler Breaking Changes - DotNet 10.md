# .NET 9.0.100 至 .NET 10.0.100 之间 Roslyn 的重大更改

本文档列出了在 .NET 9 正式发布（.NET SDK 版本 9.0.100）之后到 .NET 10 正式发布（.NET SDK 版本 10.0.100）之间，Roslyn 中已知的重大更改。

## lambda 参数列表中的 `scoped` 现在始终作为修饰符处理

***在 Visual Studio 2022 版本 17.13 中引入***

C# 14 引入了在 lambda 中使用无类型参数修饰符的能力：
[带修饰符的简单 lambda 参数](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-14.0/simple-lambda-parameters-with-modifiers.md)

作为此工作的一部分，接受了一项重大更改：`scoped` 在 lambda 参数中将始终被视为修饰符，即使它过去可能被接受为类型名称。例如：

```c#
var v = (scoped scoped s) => { ... };

ref struct @scoped { }
```

在 C# 14 中，这将是一个错误，因为两个 `scoped` 标记都被视为修饰符。解决方法是在类型名称位置使用 `@`，如下所示：

```c#
var v = (scoped @scoped s) => { ... };

ref struct @scoped { }
```

## `Span<T>` 和 `ReadOnlySpan<T>` 重载在 C# 14 及更高版本中适用于更多场景

***在 Visual Studio 2022 版本 17.13 中引入***

C# 14 引入了新的[内置 span 转换和类型推断规则](https://github.com/dotnet/csharplang/issues/7905)。
这意味着与 C# 13 相比，可能会选择不同的重载，有时还可能引发歧义编译时错误，因为新重载适用但没有单一最佳重载。

以下示例显示了一些歧义和可能的解决方法。
请注意，另一个解决方法是 API 作者使用 `OverloadResolutionPriorityAttribute`。

```csharp
var x = new long[] { 1 };
Assert.Equal([2], x); // 之前是 Assert.Equal<T>(T[], T[])，现在与 Assert.Equal<T>(ReadOnlySpan<T>, Span<T>) 存在歧义
Assert.Equal([2], x.AsSpan()); // 解决方法

var y = new int[] { 1, 2 };
var s = new ArraySegment<int>(y, 1, 1);
Assert.Equal(y, s); // 之前是 Assert.Equal<T>(T, T)，现在与 Assert.Equal<T>(Span<T>, Span<T>) 存在歧义
Assert.Equal(y.AsSpan(), s); // 解决方法
```

在 C# 14 中，可能会在 C# 13 中选择实现了 `T[]`（例如 `IEnumerable<T>`）接口的重载的位置选择 `Span<T>` 重载，如果与协变数组一起使用，可能导致运行时 `ArrayTypeMismatchException`：

```csharp
string[] s = new[] { "a" };
object[] o = s; // 数组协变

C.R(o); // 之前输出 1，现在在 Span<T> 构造函数中抛出 ArrayTypeMismatchException
C.R(o.AsEnumerable()); // 解决方法

static class C
{
    public static void R<T>(IEnumerable<T> e) => Console.Write(1);
    public static void R<T>(Span<T> s) => Console.Write(2);
    // 另一个解决方法:
    public static void R<T>(ReadOnlySpan<T> s) => Console.Write(3);
}
```

因此，在 C# 14 中，重载解析通常优先选择 `ReadOnlySpan<T>` 而非 `Span<T>`。
在某些情况下，这可能导致编译中断，
例如当同时存在 `Span<T>` 和 `ReadOnlySpan<T>` 的重载，且两者都接受并返回相同的 span 类型时：

```cs
double[] x = new double[0];
Span<ulong> y = MemoryMarshal.Cast<double, ulong>(x); // 之前有效，现在编译错误
Span<ulong> z = MemoryMarshal.Cast<double, ulong>(x.AsSpan()); // 解决方法

static class MemoryMarshal
{
    public static ReadOnlySpan<TTo> Cast<TFrom, TTo>(ReadOnlySpan<TFrom> span) => default;
    public static Span<TTo> Cast<TFrom, TTo>(Span<TFrom> span) => default;
}
```

### `Enumerable.Reverse`

当使用 C# 14 或更高版本并以早于 `net10.0` 的 .NET 或带有 `System.Memory` 引用的 .NET Framework 为目标时，
`Enumerable.Reverse` 与数组存在重大更改。

> [!CAUTION]
> 这只影响使用 C# 14 且以早于 `net10.0` 的 .NET 为目标的用户，这是不受支持的配置。

```csharp
int[] x = new[] { 1, 2, 3 };
var y = x.Reverse(); // 之前是 Enumerable.Reverse，现在是 MemoryExtensions.Reverse
```

在 `net10.0` 上，有 `Enumerable.Reverse(this T[])` 优先适用，因此可以避免此中断。
否则，将解析为 `MemoryExtensions.Reverse(this Span<T>)`，其语义与 `Enumerable.Reverse(this IEnumerable<T>)` 不同（后者在 C# 13 及以下版本中被解析）。
具体而言，`Span` 扩展方法就地执行反转并返回 `void`。
作为解决方法，可以定义自己的 `Enumerable.Reverse(this T[])` 或显式使用 `Enumerable.Reverse`：

```csharp
int[] x = new[] { 1, 2, 3 };
var y = Enumerable.Reverse(x); // 替代 'x.Reverse();'
```

## 现在会针对 `foreach` 中基于模式的释放方法报告诊断

***在 Visual Studio 2022 版本 17.13 中引入***

例如，现在会在 `await foreach` 中报告过时的 `DisposeAsync` 方法。
```csharp
await foreach (var i in new C()) { } // 'C.AsyncEnumerator.DisposeAsync()' 已过时

class C
{
    public AsyncEnumerator GetAsyncEnumerator(System.Threading.CancellationToken token = default)
    {
        throw null;
    }

    public sealed class AsyncEnumerator : System.IAsyncDisposable
    {
        public int Current { get => throw null; }
        public Task<bool> MoveNextAsync() => throw null;

        [System.Obsolete]
        public ValueTask DisposeAsync() => throw null;
    }
}
```

## 在释放期间将枚举器对象状态设置为"结束后"

***在 Visual Studio 2022 版本 17.13 中引入***

枚举器的状态机错误地允许在枚举器被释放后继续执行。
现在，对已释放枚举器调用 `MoveNext()` 将正确返回 `false`，而不会执行任何用户代码。

```csharp
var enumerator = C.GetEnumerator();

Console.Write(enumerator.MoveNext()); // 输出 True
Console.Write(enumerator.Current); // 输出 1

enumerator.Dispose();

Console.Write(enumerator.MoveNext()); // 现在输出 False

class C
{
    public static IEnumerator<int> GetEnumerator()
    {
        yield return 1;
        Console.Write("not executed after disposal")
        yield return 2;
    }
}
```

## 在简单 `or` 模式中警告冗余模式

***在 Visual Studio 2022 版本 17.13 中引入***

在析取 `or` 模式（例如 `is not null or 42` 或 `is not int or string`）中，
第二个模式是冗余的，可能是由于误解了 `not` 和 `or` 模式组合器的优先顺序。
编译器会在常见的此类错误情况下提供警告：

```csharp
_ = o is not null or 42; // 警告：模式 "42" 是冗余的
_ = o is not int or string; // 警告：模式 "string" 是冗余的
```
用户可能的意图是 `is not (null or 42)` 或 `is not (int or string)`。

## `UnscopedRefAttribute` 不能与旧的 ref 安全规则一起使用

***在 Visual Studio 2022 版本 17.13 中引入***

`UnscopedRefAttribute` 无意中影响了在较早 ref 安全规则上下文中编译的代码（即以 C# 10 或更早版本以及 net6.0 或更早版本为目标）。
但是，该属性在该上下文中不应有效，现已修复。

之前在 C# 10 或更早版本中不会报告任何错误的代码，现在可能无法编译：

```csharp
using System.Diagnostics.CodeAnalysis;
struct S
{
    public int F;

    // 之前在 C# 10 with net6.0 中允许
    // 现在失败，错误与没有 [UnscopedRef] 时相同：
    // error CS8170: 结构成员不能通过引用返回 'this' 或其他实例成员
    [UnscopedRef] public ref int Ref() => ref F;
}
```

为了防止误解（认为该属性有效，但实际上由于代码是用较早的 ref 安全规则编译的而没有效果），
当属性在 C# 10 或更早版本中与 net6.0 或更早版本一起使用时，会报告警告：

```csharp
using System.Diagnostics.CodeAnalysis;
struct S
{
    // 在 C# 10 with net6.0 中两者都是错误：
    // warning CS9269: UnscopedRefAttribute 仅在 C# 11 或更高版本中或以 net7.0 或更高版本为目标时有效。
    [UnscopedRef] public ref int Ref() => throw null!;
    public static void M([UnscopedRef] ref int x) { }
}
```

## `Microsoft.CodeAnalysis.EmbeddedAttribute` 在声明时进行验证

***在 Visual Studio 2022 版本 17.13 中引入***

编译器现在在源代码中声明 `Microsoft.CodeAnalysis.EmbeddedAttribute` 时验证其形状。之前，编译器
只有在不需要自己生成该属性时才允许用户定义此属性的声明。现在我们验证：

1. 它必须是 internal
2. 它必须是 class
3. 它必须是 sealed
4. 它必须是非静态的
5. 它必须有 internal 或 public 的无参数构造函数
6. 它必须继承自 System.Attribute
7. 它必须允许在任何类型声明上使用（class、struct、interface、enum 或 delegate）

```cs
namespace Microsoft.CodeAnalysis;

// 之前有时允许。现在，CS9271
public class EmbeddedAttribute : Attribute {}
```

## 属性访问器中的表达式 `field` 引用合成的后备字段

***在 Visual Studio 2022 版本 17.12 中引入***

在属性访问器中使用时，表达式 `field` 指代该属性的合成后备字段。

当标识符在语言版本 13 或更早版本中绑定到不同符号时，会报告警告 CS9258。

若要避免生成合成后备字段并引用现有成员，请使用 `this.field` 或 `@field`。
或者，重命名现有成员及其引用，以避免与 `field` 冲突。

```csharp
class MyClass
{
    private int field = 0;

    public object Property
    {
        get
        {
            // warning CS9258: 'field' 关键字绑定到属性的合成后备字段。
            // 若要避免生成合成后备字段并引用现有成员，
            // 请使用 'this.field' 或 '@field'。
            return field;
        }
    }
}
```

## 属性访问器中不允许名为 `field` 的变量

***在 Visual Studio 2022 版本 17.14 中引入***

在属性访问器中使用时，表达式 `field` 指代该属性的合成后备字段。

当在属性访问器中声明名为 `field` 的局部变量或嵌套函数的参数时，会报告错误 CS9272。

要避免该错误，请重命名变量，或在声明中使用 `@field`。

```csharp
class MyClass
{
    public object Property
    {
        get
        {
            // error CS9272: 'field' 是属性访问器中的关键字。
            // 请重命名变量或使用标识符 '@field'。
            int field = 0;
            return @field;
        }
    }
}
```

## `record` 和 `record struct` 类型不能定义指针类型成员，即使提供了自己的 Equals 实现

***在 Visual Studio 2022 版本 17.14 中引入***

`record class` 和 `record struct` 类型的规范表明不允许任何指针类型作为实例字段。
但是，当 `record class` 或 `record struct` 类型定义了自己的 `Equals` 实现时，这一规则没有被正确执行。

编译器现在正确地禁止了这种情况。

```cs
unsafe record struct R(
    int* P // 之前正常，现在 CS8908
)
{
    public bool Equals(R other) => true;
}
```

## 发出仅元数据的可执行文件需要入口点

***在 Visual Studio 2022 版本 17.14 中引入***

之前，在以仅元数据模式（也称为 ref 程序集）发出可执行文件时，[入口点被无意设置为未设置](https://github.com/dotnet/roslyn/issues/76707)。
现已修正，但这也意味着缺少入口点会导致编译错误：

```cs
// 之前成功，现在失败：
CSharpCompilation.Create("test").Emit(new MemoryStream(),
    options: EmitOptions.Default.WithEmitMetadataOnly(true))

CSharpCompilation.Create("test",
    // 解决方法 - 将输出类型标记为 DLL 而不是 EXE（默认值）：
    options: new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary))
    .Emit(new MemoryStream(),
        options: EmitOptions.Default.WithEmitMetadataOnly(true))
```

类似地，使用命令行参数 `/refonly` 或 MSBuild 属性 `ProduceOnlyReferenceAssembly` 时也可以观察到此行为。

## `partial` 不能作为方法的返回类型

***在 Visual Studio 2022 版本 17.14 中引入***

[分部事件和构造函数](https://github.com/dotnet/csharplang/issues/9058)语言功能
允许在更多位置使用 `partial` 修饰符，因此它不能作为返回类型，除非被转义：

```cs
class C
{
    partial F() => new partial(); // 之前有效
    @partial F() => new partial(); // 解决方法
}

class partial { }
```

## `extension` 被视为上下文关键字

***在 Visual Studio 2022 版本 17.14 中引入。***
从 C# 14 开始，`extension` 关键字在表示扩展容器时具有特殊用途。
这改变了编译器解释某些代码结构的方式。

如果需要将 "extension" 作为标识符而不是关键字使用，请使用 `@` 前缀进行转义：`@extension`。这告诉编译器将其视为常规标识符而不是关键字。

编译器将把这段代码解析为扩展容器而不是构造函数。
```csharp
class @extension
{
    extension(object o) { } // 解析为扩展容器
}
```

编译器将无法将其解析为返回类型为 `extension` 的方法。
```csharp
class @extension
{
    extension M() { } // 无法编译
}
```

***在 Visual Studio 2026 版本 18.0 中引入。***
"extension" 标识符不得用作类型名称，因此以下代码将无法编译：
```csharp
using extension = ...; // 别名不能命名为 "extension"
class extension { } // 类型不能命名为 "extension"
class C<extension> { } // 类型参数不能命名为 "extension"
```

## 分部属性和事件现在隐式为 virtual 和 public

***在 Visual Studio 2026 版本 18.0 preview 1 中引入***

我们修复了一个[不一致问题](https://github.com/dotnet/roslyn/issues/77346)：
分部接口属性和事件与其非分部等效项不同，不会隐式为 `virtual` 和 `public`。
但为避免更大的重大更改，这种不一致性对于分部接口方法[保留](./Deviations%20from%20Standard.md#interface-partial-methods)不变。
请注意，不支持默认接口成员的 Visual Basic 和其他语言将开始需要实现隐式为 virtual 的 `partial` 接口成员。

要保留之前的行为，请显式将 `partial` 接口成员标记为 `private`（如果没有任何可访问性修饰符）
和 `sealed`（如果没有暗示 `sealed` 的 `private` 修饰符，且没有修饰符 `virtual` 或 `sealed`）。

```cs
System.Console.Write(((I)new C()).P); // 之前输出 1，现在输出 2

partial interface I
{
    public partial int P { get; }
    public partial int P => 1; // 现在隐式为 virtual
}

class C : I
{
    public int P => 2; // 实现 I.P
}
```

```cs
System.Console.Write(((I)new C()).P); // 之前无法访问，现在输出 1

partial interface I
{
    partial int P { get; } // 现在隐式为 public
    partial int P => 1;
}

class C : I;
```

## `ParamCollectionAttribute` 缺失在更多情况下被报告

***在 Visual Studio 2026 版本 18.0 中引入***

如果您正在编译 `.netmodule`（注意这不适用于普通的 DLL/EXE 编译），
且有带 `params` 集合参数的 lambda 或局部函数，
且找不到 `ParamCollectionAttribute`，则现在会报告编译错误（因为该属性现在必须[发出](https://github.com/dotnet/roslyn/issues/79752)在合成方法上，
但属性类型本身不由编译器合成到 `.netmodule` 中）。
您可以通过自己定义该属性来解决此问题。

```cs
using System;
using System.Collections.Generic;
class C
{
    void M()
    {
        Func<IList<int>, int> lam = (params IList<int> xs) => xs.Count; // 如果 ParamCollectionAttribute 不存在则报错
        lam([1, 2, 3]);

        int func(params IList<int> xs) => xs.Count; // 如果 ParamCollectionAttribute 不存在则报错
        func(4, 5, 6);
    }
}
```
