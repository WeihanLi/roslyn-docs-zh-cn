# .NET 8.0.100 至 .NET 9.0.100 之间 Roslyn 的重大更改

本文档列出了在 .NET 8 正式发布（.NET SDK 版本 8.0.100）之后到 .NET 9 正式发布（.NET SDK 版本 9.0.100）之间，Roslyn 中已知的重大更改。

## record struct 类型上的 InlineArray 属性不再允许。

***在 Visual Studio 2022 版本 17.11 中引入***

```cs
[System.Runtime.CompilerServices.InlineArray(10)] // error CS9259: 属性 'System.Runtime.CompilerServices.InlineArray' 不能应用于 record struct。
record struct Buffer1()
{
    private int _element0;
}

[System.Runtime.CompilerServices.InlineArray(10)] // error CS9259: 属性 'System.Runtime.CompilerServices.InlineArray' 不能应用于 record struct。
record struct Buffer2(int p1)
{
}
```


## 在 C# 13 及更新版本中，迭代器引入安全上下文

***在 Visual Studio 2022 版本 17.11 中引入***

尽管语言规范指出迭代器引入安全上下文，但 Roslyn 在 C# 12 及更低版本中并未实现这一点。
作为[允许在迭代器中使用 unsafe 代码](https://github.com/dotnet/roslyn/issues/72662)功能的一部分，这将在 C# 13 中改变。
此更改不会破坏正常场景，因为以前就不允许直接在迭代器中使用 unsafe 构造。
但是，它可能破坏以前将 unsafe 上下文继承到嵌套局部函数中的场景，例如：

```cs
unsafe class C // unsafe 上下文
{
    System.Collections.Generic.IEnumerable<int> M() // 一个迭代器
    {
        yield return 1;
        local();
        void local()
        {
            int* p = null; // 在 C# 12 中允许；在 C# 13 中报错
        }
    }
}
```

你可以通过简单地向局部函数添加 `unsafe` 修饰符来解决这个问题。

## C# 13 及更新版本中集合表达式的重载解析重大更改

***在使用 C# 13+ 的 Visual Studio 2022 版本 17.12 及更新版本中引入***

C# 13 中集合表达式绑定有一些变化。这些变化大多数将歧义转变为成功编译，
但有几个是重大更改，会导致新的编译错误或行为重大更改。详细说明如下。

### 空集合表达式不再使用 API 是否为 span 来决定重载

当空集合表达式提供给重载方法，且没有明确的元素类型时，我们不再使用 API 是否接受 `ReadOnlySpan<T>` 或 `Span<T>` 来决定是否偏向该 API。例如：

```cs
class C
{
    static void M(ReadOnlySpan<int> ros) {}
    static void M(Span<object> s) {}

    static void Main()
    {
        M([]); // 在 C# 12 中调用 C.M(ReadOnlySpan<int>)，在 C# 13 中报错。
    }
}
```

### 精确元素类型优先于其他所有考量

在 C# 13 中，我们优先考虑精确的元素类型匹配，查看表达式的转换。这在涉及常量时可能导致行为更改：

```cs
class C
{
    static void M1(ReadOnlySpan<byte> ros) {}
    static void M1(Span<int> s) {}

    static void M2(ReadOnlySpan<string> ros) {}
    static void M2(Span<CustomInterpolatedStringHandler> ros) {}

    static void Main()
    {
        M1([1]); // 在 C# 12 中调用 C.M(ReadOnlySpan<byte>)，在 C# 13 中调用 C.M(Span<int>)

        M2([$"{1}"]); // 在 C# 12 中调用 C.M(ReadOnlySpan<string>)，在 C# 13 中调用 C.M(Span<CustomInterpolatedStringHandler>)
    }
}
```

## 在缺乏正确声明 DefaultMemberAttribute 的情况下，不再允许声明索引器。

***在 Visual Studio 2022 版本 17.13 中引入***

```cs
public interface I1
{
    public I1 this[I1 args] { get; } // error CS0656: 缺少编译器所需成员 'System.Reflection.DefaultMemberAttribute..ctor'
}
```

## 方法组自然类型中考虑默认参数和 params 参数

***在 Visual Studio 2022 版本 17.13 中引入***

以前，当使用默认参数值或 `params` 数组时，编译器会[意外地](https://github.com/dotnet/roslyn/issues/71333)
根据源代码中候选的顺序推断不同的委托类型。现在会发出歧义错误。

```cs
using System;

class Program
{
    static void Main()
    {
        var x1 = new Program().Test1; // 以前是 Action<long[]> - 现在报错
        var x2 = new Program().Test2; // 以前是匿名 void delegate(params long[]) - 现在报错

        x1();
        x2();
    }
}

static class E
{
    static public void Test1(this Program p, long[] a) => Console.Write(a.Length);
    static public void Test1(this object p, params long[] a) => Console.Write(a.Length);

    static public void Test2(this object p, params long[] a) => Console.Write(a.Length);
    static public void Test2(this Program p, long[] a) => Console.Write(a.Length);
}
```

此外在 `LangVersion=12` 或更低版本中，`params` 修饰符在所有方法中必须匹配才能推断唯一的委托签名。
注意，这不影响 `LangVersion=13` 及更高版本，因为有[不同的委托推断算法](https://github.com/dotnet/csharplang/issues/7429)。

```cs
var d = new C().M; // 以前推断为 Action<int[]> - 现在 error CS8917: 无法推断委托类型

static class E
{
    public static void M(this C c, params int[] x) { }
}

class C
{
    public void M(int[] x) { }
}
```

解决方法是在这些情况下使用显式委托类型，而不是依赖 `var` 推断。

## `dotnet_style_require_accessibility_modifiers` 现在一致地应用于接口成员

PR: https://github.com/dotnet/roslyn/pull/76324

在此更改之前，`dotnet_style_require_accessibility_modifiers` 的分析器会简单地忽略接口成员。这是因为 C# 最初完全禁止接口成员的修饰符，让它们始终是 public 的。

语言的后续版本放宽了这一限制，允许用户在接口成员上提供可访问性修饰符，包括冗余的 `public` 修饰符。

分析器已更新，现在也会对接口成员强制执行此选项的值。该值的含义如下：

1. `never`。分析器不进行分析。所有成员上允许冗余修饰符。
2. `always`。所有成员上始终需要冗余修饰符（包括接口成员）。例如：类成员上的 `private` 修饰符，以及接口成员上的 `public` 修饰符。如果你认为所有成员都应该显式声明其可访问性，请使用此选项。
4. `for_non_interface_members`。所有*非*接口成员上需要冗余修饰符，但接口成员不允许使用。例如：`private` 在私有类成员上是必需的，但公共接口成员不允许使用冗余的 `public` 修饰符。这匹配了在语言允许接口成员使用修饰符之前的标准修饰符方法。
5. `omit_if_default`。不允许冗余修饰符。例如，私有类成员不允许使用 `private`，公共接口成员不允许使用 `public`。如果你认为在可访问性与语言默认选择相匹配时重申可访问性是多余的，请使用此选项。
