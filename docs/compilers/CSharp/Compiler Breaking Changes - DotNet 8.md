# .NET 7.0.100 至 .NET 8.0.100 之间 Roslyn 的重大更改

本文档列出了在 .NET 7 正式发布（.NET SDK 版本 7.0.100）之后到 .NET 8 正式发布（.NET SDK 版本 8.0.100）之间，Roslyn 中已知的重大更改。

## dynamic 参数的 ref 修饰符应与对应参数的 ref 修饰符兼容

***在 Visual Studio 2022 版本 17.10 中引入***

dynamic 参数的 ref 修饰符在编译时应与对应参数的 ref 修饰符兼容。这可能导致涉及 dynamic 参数的重载解析在编译时失败，而不是在运行时失败。

以前，编译时允许不匹配，将重载解析失败推迟到运行时。

例如，以下代码以前编译时没有错误，但在运行时失败，抛出异常："Microsoft.CSharp.RuntimeBinder.RuntimeBinderException: 'C.f(ref object)' 的最佳重载方法匹配有一些无效参数"。
现在它将产生编译错误。
```csharp
public class C
{
    public void f(ref dynamic a) 
    {
    }
    
    public void M(dynamic d)
    {
        f(d); // error CS1620: 参数 1 必须使用 'ref' 关键字传递
    }
}
```

## 为实现 `IEnumerable` 的类型编写的集合表达式，其元素必须可隐式转换为 `object`

***在 Visual Studio 2022 版本 17.10 中引入***

将集合表达式*转换*为实现 `System.Collections.IEnumerable` 且*没有*强类型 `GetEnumerator()` 的 `struct` 或 `class`，
需要集合表达式中的元素可隐式转换为 `object`。
以前，针对 `IEnumerable` 实现的集合表达式的元素被假定为可转换为 `object`，仅在绑定到适用的 `Add` 方法时才进行转换。

这个额外要求意味着对 `IEnumerable` 实现的集合表达式转换，与其他目标类型（元素必须可隐式转换为目标类型的*迭代类型*）保持一致。

此更改影响针对 `IEnumerable` 实现的集合表达式，其中元素依赖于目标类型来确定强类型的 `Add` 方法参数类型。
在以下示例中，报告了 `_ => { }` 无法隐式转换为 `object` 的错误。
```csharp
class Actions : IEnumerable
{
    public void Add(Action<int> action);
    // ...
}

Actions a = [_ => { }]; // error CS8917: 无法推断委托类型。
```

要解决此错误，元素表达式可以显式类型化。
```csharp
a = [(int _) => { }];          // 正常
a = [(Action<int>)(_ => { })]; // 正常
```

## 集合表达式目标类型必须有构造函数和 `Add` 方法

***在 Visual Studio 2022 版本 17.10 中引入***

将集合表达式*转换*为实现 `System.Collections.IEnumerable` 且*没有* `CollectionBuilderAttribute` 的 `struct` 或 `class`，
需要目标类型有可以无参数调用的可访问构造函数，以及如果集合表达式不为空，目标类型必须有可以用单个参数调用的可访问 `Add` 方法。

以前，构造函数和 `Add` 方法对于集合实例的*构造*是必需的，但对于*转换*不是必需的。
这意味着以下调用以前是歧义的，因为 `char[]` 和 `string` 都是集合表达式的有效目标类型。
调用不再是歧义的，因为 `string` 没有无参数构造函数或 `Add` 方法。
```csharp
Print(['a', 'b', 'c']); // 调用 Print(char[])

static void Print(char[] arg) { }
static void Print(string arg) { }
```

## `ref` 参数可以传递给 `in` 参数

***在 Visual Studio 2022 版本 17.8p2 中引入***

[`ref readonly` 参数](https://github.com/dotnet/csharplang/issues/6010)功能在 `LangVersion` 设置为 12 或更高版本时放宽了重载解析，允许将 `ref` 参数传递给 `in` 参数。
这可能导致行为或源代码重大更改：

```cs
var i = 5;
System.Console.Write(new C().M(ref i)); // 在 C# 11 中输出 "E"，在 C# 12 中输出 "C"
System.Console.Write(E.M(new C(), ref i)); // 解决方法：始终输出 "E"

class C
{
    public string M(in int i) => "C";
}
static class E
{
    public static string M(this C c, ref int i) => "E";
}
```

```cs
var i = 5;
System.Console.Write(C.M(null, ref i)); // 在 C# 11 中输出 "1"，但在 C# 12 中因歧义错误而失败
System.Console.Write(C.M((I1)null, ref i)); // 解决方法：始终输出 "1"

interface I1 { }
interface I2 { }
static class C
{
    public static string M(I1 o, ref int x) => "1";
    public static string M(I2 o, in int x) => "2";
}
```

## 在异步 `using` 中优先使用基于模式的释放而不是接口释放

***在 Visual Studio 2022 版本 17.10p3 中引入***

异步 `using` 优先使用基于模式的 `DisposeAsync()` 方法，而不是接口的 `IAsyncDisposable.DisposeAsync()`。

例如，公共 `DisposeAsync()` 方法将被选择，而不是私有接口实现：
```csharp
await using (var x = new C()) { }

public class C : System.IAsyncDisposable
{
    ValueTask IAsyncDisposable.DisposeAsync() => throw null; // 不再被选择

    public async ValueTask DisposeAsync()
    {
        Console.WriteLine("PICKED");
        await Task.Yield();
    }
}
```
