# .NET 6.0.100 至 .NET 7.0.100 之间 Roslyn 的重大更改

本文档列出了在 .NET 6 正式发布（.NET SDK 版本 6.0.100）之后到 .NET 7 正式发布（.NET SDK 版本 7.0.100）之间，Roslyn 中已知的重大更改。

## 异步方法中所有受限类型的局部变量均被禁止

***在 Visual Studio 2022 版本 17.6p1 中引入***

受限类型的局部变量在异步方法中是禁止的。但在早期版本中，编译器未能注意到某些隐式声明的局部变量。例如在 `foreach`、`using` 语句或解构中。
现在，这类隐式声明的局部变量同样被禁止。

```csharp
ref struct RefStruct { public void Dispose() { } }
public class C 
{
    public async Task M() 
    {
        RefStruct local = default; // 不允许
        using (default(RefStruct)) { } // 现在也不允许（"error CS9104: 此类型的 using 语句资源不能用于异步方法或异步 lambda 表达式"）
    }
}
```

参见 https://github.com/dotnet/roslyn/pull/66264

## 指针必须始终在 unsafe 上下文中。

***在 Visual Studio 2022 版本 17.6 中引入***

在早期 SDK 中，编译器偶尔会允许在没有明确标记为 unsafe 的位置引用指针。
现在，必须有 `unsafe` 修饰符。
例如 `using Alias = List<int*[]>;` 应改为 `using unsafe Alias = List<int*[]>;` 才合法。
用法如 `void Method(Alias a) ...` 应改为 `unsafe void Method(Alias a) ...`。

该规则是无条件的，但 `using` 别名声明除外（在 C# 12 之前不允许使用 `unsafe` 修饰符）。
因此对于 `using` 声明，该规则仅在选择 C# 12 或更高语言版本时生效。

## System.TypedReference 被视为托管类型

***在 Visual Studio 2022 版本 17.6 中引入***

此后，`System.TypedReference` 类型被视为托管类型。

```csharp
unsafe
{
    TypedReference* r = null; // 警告：对托管类型取地址、获取大小或声明指针
    var a = stackalloc TypedReference[1]; // 错误：无法对托管类型取地址、获取大小或声明指针
}
```

## ref 安全错误不影响 lambda 表达式到委托的转换

***在 Visual Studio 2022 版本 17.5 中引入***

lambda 主体中报告的 ref 安全错误不再影响 lambda 表达式是否可转换为委托类型。这一变化可能影响重载解析。

在以下示例中，使用 Visual Studio 17.5 调用 `M(x => ...)` 存在歧义，因为 `M(D1)` 和 `M(D2)` 现在都被认为是适用的，即使 lambda 主体中对 `F(ref x, ref y)` 的调用在使用 `M(D1)` 时会导致 ref 安全错误（参见 `d1` 和 `d2` 的示例进行比较）。以前，该调用会明确绑定到 `M(D2)`，因为 `M(D1)` 重载被认为不适用。
```csharp
using System;

ref struct R { }

delegate R D1(R r);
delegate object D2(object o);

class Program
{
    static void M(D1 d1) { }
    static void M(D2 d2) { }

    static void F(ref R x, ref Span<int> y) { }
    static void F(ref object x, ref Span<int> y) { }

    static void Main()
    {
        // error CS0121: 'M(D1)' 和 'M(D2)' 之间存在歧义
        M(x =>
            {
                Span<int> y = stackalloc int[1];
                F(ref x, ref y);
                return x;
            });

        D1 d1 = x1 =>
            {
                Span<int> y1 = stackalloc int[1];
                F(ref x1, ref y1); // error CS8352: 'y2' 可能会将参数引用的变量暴露在其声明范围之外
                return x1;
            };

        D2 d2 = x2 =>
            {
                Span<int> y2 = stackalloc int[1];
                F(ref x2, ref y2); // 正常: F(ref object x, ref Span<int> y)
                return x2;
            };
    }
}
```

要解决重载解析的变化，可以为 lambda 参数或委托使用显式类型。

```csharp
        // 正常: M(D2)
        M((object x) =>
            {
                Span<int> y = stackalloc int[1];
                F(ref x, ref y); // 正常: F(ref object x, ref Span<int> y)
                return x;
            });
```

## 行首的原始字符串插值。

***在 Visual Studio 2022 版本 17.5 中引入***

在 .NET SDK 7.0.100 或更早版本中，以下代码被错误地允许：

```csharp
var x = $"""
    Hello
{1 + 1}
    World
    """;
```

这违反了行内容（包括插值开始的位置）必须以与最终 `    """;` 行相同的空白开头的规则。现在要求将上述内容写成：


```csharp
var x = $"""
    Hello
    {1 + 1}
    World
    """;
```


## 方法的推断委托类型包含默认参数值和 `params` 修饰符

***在 Visual Studio 2022 版本 17.5 中引入***

在 .NET SDK 7.0.100 或更早版本中，从方法推断的委托类型忽略了默认参数值和 `params` 修饰符，如以下代码所示：

```csharp
void Method(int i = 0, params int[] xs) { }
var action = Method; // System.Action<int, int[]>
DoAction(action, 1); // 正常
void DoAction(System.Action<int, int[]> a, int p) => a(p, new[] { p });
```

在 .NET SDK 7.0.200 或更高版本中，此类方法被推断为具有相同默认参数值和 `params` 修饰符的匿名合成委托类型。
这一变化会破坏上述代码，如下所示：

```csharp
void Method(int i = 0, params int[] xs) { }
var action = Method; // delegate void <anonymous delegate>(int arg1 = 0, params int[] arg2)
DoAction(action, 1); // error CS1503: 参数 1：无法从 '<anonymous delegate>' 转换为 'System.Action<int, int[]>'
void DoAction(System.Action<int, int[]> a, int p) => a(p, new[] { p });
```

你可以在相关[提案](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/lambda-method-group-defaults.md#breaking-change)中了解更多关于此更改的信息。

## 对于确定赋值分析，异步局部函数的调用不再被视为已等待

***在 Visual Studio 2022 版本 17.5 中引入***

对于确定赋值分析，异步局部函数的调用不再被视为已等待，因此局部函数不被认为已完全执行。参见 https://github.com/dotnet/roslyn/issues/43697 了解原因。

以下代码现在将报告确定赋值错误：
```csharp
    public async Task M()
    {
        bool a;
        await M1();
        Console.WriteLine(a); // error CS0165: 使用了未赋值的局部变量 'a'

        async Task M1()
        {
            if ("" == String.Empty)
            {
                throw new Exception();
            }
            else
            {
                a = true;
            }
        }
    }
```

## 属性的 `INoneOperation` 节点现在是 `IAttributeOperation` 节点。

***在 Visual Studio 2022 版本 17.5、.NET SDK 版本 7.0.200 中引入***

在早期版本的编译器中，属性的 `IOperation` 树以 `INoneOperation` 节点为根。
我们添加了对属性的原生支持，这意味着树的根现在是 `IAttributeOperation`。一些分析器，包括较旧版本的 .NET SDK 分析器，不期望这种树形状，在遇到时会不正确地警告（或可能无法警告）。解决方法是：

* 如果可能，更新分析器版本。如果使用 .NET SDK 或旧版本的 Microsoft.CodeAnalysis.FxCopAnalyzers，请更新到 Microsoft.CodeAnalysis.NetAnalyzers 7.0.0-preview1.22464.1 或更新版本。
* 对分析器的误报进行抑制，直到它们可以通过考虑此更改的版本进行更新。

## 不支持对 `ref` struct 进行类型测试。

***在 Visual Studio 2022 版本 17.4 中引入***

当 `ref` struct 类型用于 'is' 或 'as' 运算符时，在某些情况下编译器以前会错误地报告类型测试在运行时始终失败的警告，省略实际的类型检查，并导致不正确的行为。当运行时的不正确行为是可能的时，编译器现在将产生错误。

```csharp
ref struct G<T>
{
    public void Test()
    {
        if (this is G<int>) // 现在将产生错误，以前被视为始终为 `false`。
        {
```

## 对 ref 局部变量的未使用结果进行解引用。

***在 Visual Studio 2022 版本 17.4 中引入***

当 `ref` 局部变量通过值被引用，但结果未被使用（例如被赋值给丢弃）时，结果以前被忽略。编译器现在将对该局部变量进行解引用，确保观察到任何副作用。

```csharp
ref int local = Unsafe.NullRef<int>();
_ = local; // 现在将产生 `NullReferenceException`
```

## 类型不能命名为 `scoped`

***在 Visual Studio 2022 版本 17.4 中引入。*** 从 C# 11 开始，类型不能命名为 `scoped`。编译器将对所有此类类型名称报告错误。要解决此问题，类型名称及所有用法必须使用 `@` 进行转义：

```csharp
class scoped {} // Error CS9056
class @scoped {} // 无错误
```

```csharp
ref scoped local; // 错误
ref scoped.nested local; // 错误
ref @scoped local2; // 无错误
```

这是因为 `scoped` 现在是变量声明的修饰符，在 ref 类型中跟在 `ref` 后面时是保留的。

## 类型不能命名为 `file`

***在 Visual Studio 2022 版本 17.4 中引入。*** 从 C# 11 开始，类型不能命名为 `file`。编译器将对所有此类类型名称报告错误。要解决此问题，类型名称及所有用法必须使用 `@` 进行转义：

```csharp
class file {} // Error CS9056
class @file {} // 无错误
```

这是因为 `file` 现在是类型声明的修饰符。

你可以在相关的 [csharplang issue](https://github.com/dotnet/csharplang/issues/6011) 中了解更多关于此更改的信息。

## #line span 指令中所需的空格

***在 .NET SDK 6.0.400、Visual Studio 2022 版本 17.3 中引入。***

当 `#line` span 指令在 C# 10 中引入时，它不需要特定的间距。
例如，这是有效的：`#line(1,2)-(3,4)5"file.cs"`。

在 Visual Studio 17.3 中，编译器要求在第一个括号、字符偏移量和文件名之前有空格。
因此，上述示例除非添加空格，否则无法解析：`#line (1,2)-(3,4) 5 "file.cs"`。

## System.IntPtr 和 System.UIntPtr 上的 checked 运算符

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.3 中引入。***

当平台支持__数值__ `IntPtr` 和 `UIntPtr` 类型时（由 `System.Runtime.CompilerServices.RuntimeFeature.NumericIntPtr` 的存在指示），`nint` 和 `nuint` 的内置运算符适用于这些底层类型。
这意味着在此类平台上，`IntPtr` 和 `UIntPtr` 具有内置的 `checked` 运算符，在发生溢出时现在可能会抛出异常。

```csharp
IntPtr M(IntPtr x, int y)
{
    checked
    {
        return x + y; // 现在可能会抛出
    }
}

unsafe IntPtr M2(void* ptr)
{
    return checked((IntPtr)ptr); // 现在可能会抛出
}
```

可能的解决方法是：

1. 指定 `unchecked` 上下文
2. 降级到不支持数值 `IntPtr`/`UIntPtr` 类型的平台/TFM

此外，在此类平台上，`IntPtr`/`UIntPtr` 与其他数值类型之间的隐式转换被视为标准转换。这在某些情况下可能影响重载解析。

## System.UIntPtr 和 System.Int32 的加法

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.3 中引入。***

当平台支持__数值__ `IntPtr` 和 `UIntPtr` 类型时，`System.UIntPtr` 中定义的 `+(UIntPtr, int)` 运算符不再可用。
相反，对 `System.UIntPtr` 和 `System.Int32` 类型表达式进行加法会产生错误：

```csharp
UIntPtr M(UIntPtr x, int y)
{
    return x + y; // error: 运算符 '+' 对 'nuint' 和 'int' 类型的操作数不明确
}
```

可能的解决方法是：

1. 使用 `UIntPtr.Add(UIntPtr, int)` 方法：`UIntPtr.Add(x, y)`
2. 对第二个操作数应用 unchecked 转型为 `nuint` 类型：`x + unchecked((nuint)y)`

## 方法或局部函数上的属性中的 nameof 运算符

***在 .NET SDK 6.0.400、Visual Studio 2022 版本 17.3 中引入。***

当语言版本为 C# 11 或更高版本时，方法上的属性中的 `nameof` 运算符会将该方法的类型参数引入作用域。这同样适用于局部函数。
方法、其类型参数或参数上的属性中的 `nameof` 运算符会将该方法的参数引入作用域。这同样适用于局部函数、lambda、委托和索引器。

例如，以下代码现在会产生错误：
```csharp
class C
{
  class TParameter
  {
    internal const string Constant = """";
  }
  [MyAttribute(nameof(TParameter.Constant))]
  void M<TParameter>() { }
}
```

```csharp
class C
{
  class parameter
  {
    internal const string Constant = """";
  }
  [MyAttribute(nameof(parameter.Constant))]
  void M(int parameter) { }
}
```

可能的解决方法是：

1. 重命名类型参数或参数以避免遮蔽外部作用域的名称。
1. 使用字符串字面量代替 `nameof` 运算符。

## 不能通过引用返回 out 参数

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.3 中引入。***

使用 C# 11 或更高语言版本，或使用 .NET 7.0 或更高版本时，`out` 参数不能通过引用返回。

```csharp
static ref T ReturnOutParamByRef<T>(out T t)
{
    t = default;
    return ref t; // error CS8166: 无法通过引用返回参数 't'，因为它不是 ref 参数
}
```

可能的解决方法是：
1. 使用 `System.Diagnostics.CodeAnalysis.UnscopedRefAttribute` 将引用标记为无范围。
    ```csharp
    static ref T ReturnOutParamByRef<T>([UnscopedRef] out T t)
    {
        t = default;
        return ref t; // 正常
    }
    ```

1. 将方法签名更改为通过 `ref` 传递参数。
    ```csharp
    static ref T ReturnRefParamByRef<T>(ref T t)
    {
        t = default;
        return ref t; // 正常
    }
    ```

## ref struct 上的实例方法可能捕获无范围的 ref 参数

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.4 中引入。***

使用 C# 11 或更高语言版本，或使用 .NET 7.0 或更高版本时，`ref struct` 实例方法调用被假定为捕获无范围的 `ref` 或 `in` 参数。

```csharp
R<int> Use(R<int> r)
{
    int i = 42;
    r.MayCaptureArg(ref i); // error CS8350: 可能会将参数 't' 引用的变量暴露在其声明范围之外
    return r;
}

ref struct R<T>
{
    public void MayCaptureArg(ref T t) { }
}
```

如果 `ref` 或 `in` 参数没有在 `ref struct` 实例方法中被捕获，一个可能的解决方法是将参数声明为 `scoped ref` 或 `scoped in`。

```csharp
R<int> Use(R<int> r)
{
    int i = 42;
    r.CannotCaptureArg(ref i); // 正常
    return r;
}

ref struct R<T>
{
    public void CannotCaptureArg(scoped ref T t) { }
}
```

## 方法 ref struct 返回转义分析依赖于 ref 参数的 ref 转义

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.4 中引入。***

使用 C# 11 或更高语言版本，或使用 .NET 7.0 或更高版本时，从方法调用返回的 `ref struct`（作为返回值或 `out` 参数）仅在所有 `ref` 和 `in` 参数都是 *ref-safe-to-escape* 时才是*safe-to-escape*的。*`in` 参数可能包括隐式默认参数值。*

```csharp
ref struct R { }

static R MayCaptureArg(ref int i) => new R();

static R MayCaptureDefaultArg(in int i = 0) => new R();

static R Create()
{
    int i = 0;
    // error CS8347: 无法使用 'MayCaptureArg(ref int)' 的结果，因为它可能会将参数 'i' 引用的变量暴露在其声明范围之外
    return MayCaptureArg(ref i);
}

static R CreateDefault()
{
    // error CS8347: 无法使用 'MayCaptureDefaultArg(in int)' 的结果，因为它可能会将参数 'i' 引用的变量暴露在其声明范围之外
    return MayCaptureDefaultArg();
}
```

如果 `ref` 或 `in` 参数没有在 `ref struct` 返回值中被捕获，一个可能的解决方法是将参数声明为 `scoped ref` 或 `scoped in`。

```csharp
static R CannotCaptureArg(scoped ref int i) => new R();

static R Create()
{
    int i = 0;
    return CannotCaptureArg(ref i); // 正常
}
```

## `__arglist` 中对 ref struct 参数的 `ref` 被认为是无范围的

***在 .NET SDK 7.0.100、Visual Studio 2022 版本 17.4 中引入。***

使用 C# 11 或更高语言版本，或使用 .NET 7.0 或更高版本时，当作为参数传递给 `__arglist` 时，对 `ref struct` 类型的 `ref` 被认为是无范围引用。

```csharp
ref struct R { }

class Program
{
    static void MayCaptureRef(__arglist) { }

    static void Main()
    {
        var r = new R();
        MayCaptureRef(__arglist(ref r)); // 错误：可能会将变量暴露在其声明范围之外
    }
}
```

## 无符号右移运算符

***在 .NET SDK 6.0.400、Visual Studio 2022 版本 17.3 中引入。***
语言新增了对"无符号右移"运算符（`>>>`）的支持。
这禁止了将实现用户定义的"无符号右移"运算符的方法作为常规方法调用。

例如，存在一个用某种语言（非 VB 或 C#）开发的现有库，它为类型 ```C1``` 公开了"无符号右移"用户定义运算符。
以下代码以前可以成功编译：
``` C#
static C1 Test1(C1 x, int y) => C1.op_UnsignedRightShift(x, y); //error CS0571: 'C1.operator >>>(C1, int)': 无法显式调用运算符或访问器
``` 

一个可能的解决方法是改用 `>>>` 运算符：
``` C#
static C1 Test1(C1 x, int y) => x >>> y;
``` 

## foreach 枚举器作为 ref struct

***在 .NET SDK 6.0.300、Visual Studio 2022 版本 17.2 中引入。*** 使用 ref struct 枚举器类型的 `foreach` 在语言版本设置为 7.3 或更早时会报告错误。

这修复了一个 bug，该 bug 导致在针对 C# 版本时，新编译器在支持该功能之前就支持了该特性。

可能的解决方法是：

1. 将 `ref struct` 类型改为 `struct` 或 `class` 类型。
1. 将 `<LangVersion>` 元素升级到 7.3 或更高版本。

## 异步 `foreach` 优先使用基于模式的 `DisposeAsync` 而不是 `IAsyncDisposable.DisposeAsync()` 的显式接口实现

***在 .NET SDK 6.0.300、Visual Studio 2022 版本 17.2 中引入。*** 异步 `foreach` 优先使用基于模式的 `DisposeAsync()` 方法，而不是 `IAsyncDisposable.DisposeAsync()`。

例如，将选择 `DisposeAsync()` 而不是 `AsyncEnumerator` 上的 `IAsyncEnumerator<int>.DisposeAsync()` 方法：

```csharp
await foreach (var i in new AsyncEnumerable())
{
}

struct AsyncEnumerable
{
    public AsyncEnumerator GetAsyncEnumerator() => new AsyncEnumerator();
}

struct AsyncEnumerator : IAsyncDisposable
{
    public int Current => 0;
    public async ValueTask<bool> MoveNextAsync()
    {
        await Task.Yield();
        return false;
    }
    public async ValueTask DisposeAsync()
    {
        Console.WriteLine("PICKED");
        await Task.Yield();
    }
    ValueTask IAsyncDisposable.DisposeAsync() => throw null; // 不再被选择
}
```

此更改修复了一个规范违规，即声明类型上的公共 `DisposeAsync` 方法是可见的，而显式接口实现只能通过接口类型的引用访问。

要解决此错误，请从类型中删除基于模式的 `DisposeAsync` 方法。

## 不允许将转换的字符串作为默认参数

***在 .NET SDK 6.0.300、Visual Studio 2022 版本 17.2 中引入。*** C# 编译器会接受涉及字符串常量引用转换的不正确默认参数值，并会发出 `null` 作为常量值，而不是源代码中指定的默认值。在 Visual Studio 17.2 中，这变成了错误。参见 [roslyn#59806](https://github.com/dotnet/roslyn/pull/59806)。

此更改修复了编译器中的规范违规。默认参数必须是编译时常量。以前的版本允许以下代码：

```csharp
void M(IEnumerable<char> s = "hello")
```

上述声明需要从 `string` 到 `IEnumerable<char>` 的转换。编译器允许了此构造，并会发出 `null` 作为参数的值。从 17.2 开始，上述代码产生编译器错误。

要解决此更改，你可以进行以下修改之一：

1. 更改参数类型，使转换不再是必需的。
1. 将默认参数的值更改为 `null` 以恢复以前的行为。

## 上下文关键字 `var` 作为显式 lambda 返回类型

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** 上下文关键字 var 不能用作显式 lambda 返回类型。

此更改通过确保 `var` 保持 lambda 表达式返回类型的自然类型来支持[潜在的未来功能](https://github.com/dotnet/csharplang/blob/main/meetings/2021/LDM-2021-06-02.md#lambda-return-type-parsing)。

如果你有一个名为 `var` 的类型，并且使用 `var`（类型）的显式返回类型定义 lambda 表达式，则可能会遇到此错误。

```csharp
using System;

F(var () => default);  // error CS8975: 上下文关键字 'var' 不能用作显式 lambda 返回类型
F(@var () => default); // 正常
F(() => default);      // 正常：返回类型从 F() 的参数推断

static void F(Func<var> f) { }

public class var
{
}
```

解决方法包括以下更改：

1. 使用 `@var` 作为返回类型。
1. 删除显式返回类型，让编译器确定返回类型。

## 插值字符串处理器和索引器初始化

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** 接受插值字符串处理器并需要接收器作为构造函数输入的索引器不能用于对象初始化器中。

此更改禁止了一个边缘案例场景，其中索引器初始化器使用插值字符串处理器，而该插值字符串处理器将索引器的接收器作为构造函数的参数。此更改的原因是此场景可能导致访问尚未初始化的变量。考虑以下示例：

```csharp
using System.Runtime.CompilerServices;

// 错误：引用被索引实例的插值字符串处理器转换不能用于索引器成员初始化器中。
var c = new C { [$""] = 1 }; 

class C
{
    public int this[[InterpolatedStringHandlerArgument("")] CustomHandler c]
    {
        get => ...;
        set => ...;
    }
}

[InterpolatedStringHandler]
class CustomHandler
{
    // 字符串处理器的构造函数接受 "C" 实例：
    public CustomHandler(int literalLength, int formattedCount, C c) {}
}
```

解决方法包括以下更改：

1. 从插值字符串处理器中删除接收器类型。
1. 将索引器的参数类型改为 `string`。

## 带有 UnmanagedCallersOnly 的方法上的 ref、readonly ref、in、out 不允许用作参数或返回值

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** `ref`/`ref readonly`/`in`/`out` 不允许用于标有 `UnmanagedCallersOnly` 的方法的返回值/参数。

此更改是一个 bug 修复。返回值和参数不是可直接复制的。通过引用传递参数或返回值可能导致未定义行为。以下声明均不能编译：

```csharp
using System.Runtime.InteropServices;
[UnmanagedCallersOnly]
static ref int M1() => throw null; // error CS8977: 无法在标有 'UnmanagedCallersOnly' 的方法中使用 'ref'、'in' 或 'out'。

[UnmanagedCallersOnly]
static ref readonly int M2() => throw null; // error CS8977: 无法在标有 'UnmanagedCallersOnly' 的方法中使用 'ref'、'in' 或 'out'。

[UnmanagedCallersOnly]
static void M3(ref int o) => throw null; // error CS8977: 无法在标有 'UnmanagedCallersOnly' 的方法中使用 'ref'、'in' 或 'out'。

[UnmanagedCallersOnly]
static void M4(in int o) => throw null; // error CS8977: 无法在标有 'UnmanagedCallersOnly' 的方法中使用 'ref'、'in' 或 'out'。

[UnmanagedCallersOnly]
static void M5(out int o) => throw null; // error CS8977: 无法在标有 'UnmanagedCallersOnly' 的方法中使用 'ref'、'in' 或 'out'。
```

解决方法是删除引用修饰符。

## 模式中假定 Length、Count 为非负数

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** 可数和可索引类型上的 `Length` 和 `Count` 属性
被假定为非负数，用于模式和 switch 的包含性和穷举性分析。
这些类型可以与隐式 Index 索引器和列表模式一起使用。

`Length` 和 `Count` 属性，即使类型为 `int`，在分析模式时也被假定为非负数。考虑以下示例方法：

```csharp
string SampleSizeMessage<T>(IList<T> samples)
{
    return samples switch
    {
        // 此 switch 分支在 17.1 之前可防止警告，但在实践中永远不会发生。
        // 从 17.1 开始，此 switch 分支产生编译器错误。
        // 删除它不会引入警告。
        { Count: < 0 }    => throw new InvalidOperationException(),
        { Count:  0 }     => "空集合",
        { Count: < 5 }    => "太小",
        { Count: < 20 }   => "第一次合理",
        { Count: < 100 }  => "合理",
        { Count: >= 100 } => "好",
    };
}

void M(int[] i)
{
    if (i is { Length: -1 }) {} // 错误：假定长度为非负数，这是不可能的
}
```

在 17.1 之前，测试 `Count` 为负数的第一个 switch 分支是必要的，以避免未覆盖所有可能值的警告。从 17.1 开始，第一个 switch 分支生成编译器错误。解决方法是删除为无效情况添加的 switch 分支。

## <a name="6"></a> 向 struct 添加字段初始化器需要显式声明的构造函数

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** 带有字段初始化器的 `struct` 类型声明必须包含显式声明的构造函数。此外，所有字段必须在没有 `: this()` 初始化器的 `struct` 实例构造函数中被确定赋值，因此任何以前未赋值的字段必须从添加的构造函数或字段初始化器中赋值。参见 [dotnet/csharplang#5552](https://github.com/dotnet/csharplang/issues/5552)、[dotnet/roslyn#58581](https://github.com/dotnet/roslyn/pull/58581)。

在 C# 中有两种将变量初始化为默认值的方法：`new()` 和 `default`。对于类，区别很明显，因为 `new` 创建一个新实例，`default` 返回 `null`。对于 struct，区别更微妙，因为对于 `default`，struct 返回每个字段/属性设置为其自身默认值的实例。我们在 C# 10 中为 struct 添加了字段初始化器。字段初始化器仅在显式声明的构造函数运行时执行。重要的是，它们在你使用 `default` 或创建任何 `struct` 类型的数组时不会执行。

在 17.0 中，如果有字段初始化器但没有声明的构造函数，则会合成一个运行字段初始化器的无参构造函数。然而，这意味着添加或删除构造函数声明可能会影响是否合成无参构造函数，从而可能改变 `new()` 的行为。

为了解决这个问题，在 .NET SDK 6.0.200（VS 17.1）中，编译器不再合成无参构造函数。如果 `struct` 包含字段初始化器且没有显式构造函数，编译器会生成错误。如果 `struct` 有字段初始化器，它必须声明一个构造函数，否则字段初始化器永远不会执行。

此外，所有没有字段初始化器的字段必须在每个 `struct` 构造函数中赋值，除非构造函数有 `: this()` 初始化器。

例如：

```csharp
struct S // error CS8983: 带有字段初始化器的 'struct' 必须包含显式声明的构造函数。
{
    int X = 1; 
    int Y;
}
```

解决方法是声明一个构造函数。除非字段以前未赋值，否则此构造函数可以（且通常会）是一个空的无参构造函数：

```csharp
struct S
{
    int X = 1;
    int Y;

    public S() { Y = 0; } // 正常
}
```

## 格式说明符不能包含大括号

***在 .NET SDK 6.0.200、Visual Studio 2022 版本 17.1 中引入。*** 插值字符串中的格式说明符不能包含大括号（`{` 或 `}`）。在以前的版本中，`{{` 被解释为转义的 `{`，`}}` 被解释为格式说明符中的转义 `}` 字符。现在，格式说明符中的第一个 `}` 字符结束插值，任何 `{` 字符都是错误。

这使插值字符串处理与 `System.String.Format` 的处理保持一致：

```csharp
using System;
Console.WriteLine($"{{{12:X}}}");
//现在输出: "{C}" - 而不是 "{X}}"
```

`X` 是大写十六进制的格式，`C` 是 12 的十六进制值。

解决方法是删除格式字符串中多余的大括号。

你可以在相关的 [roslyn issue](https://github.com/dotnet/roslyn/issues/57750) 中了解更多关于此更改的信息。

## 类型不能命名为 `required`

***在 Visual Studio 2022 版本 17.3 中引入。*** 从 C# 11 开始，类型不能命名为 `required`。编译器将对所有此类类型名称报告错误。要解决此问题，类型名称及所有用法必须使用 `@` 进行转义：

```csharp
class required {} // Error CS9029
class @required {} // 无错误
```

这是因为 `required` 现在是属性和字段的成员修饰符。

你可以在相关的 [csharplang issue](https://github.com/dotnet/csharplang/issues/3630) 中了解更多关于此更改的信息。
