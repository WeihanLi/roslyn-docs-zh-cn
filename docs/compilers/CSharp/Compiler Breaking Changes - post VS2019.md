## 本文档列出了 *Visual Studio 2019 Update 1* 及以后版本相对于 *Visual Studio 2019* 在 Roslyn 中的已知重大更改。

*重大更改按单调递增的编号列表格式化，以便通过简写引用（即"已知重大更改 #1"）。
每个条目应包括简短的更改描述，后跟描述更改完整详情的链接或内联的完整详情。*

1. https://github.com/dotnet/roslyn/issues/34882 C# `8.0` 中的新功能将允许对开放类型使用常量模式。例如，以下代码将被允许：
    ``` c#
    bool M<T>(T t) => t is null;
    ```
    但是，在 *Visual Studio 2019* 中，我们错误地允许在语言版本 `7.0`、`7.1`、`7.2` 和 `7.3` 中编译这段代码。在 *Visual Studio 2019 Update 1* 中，这将成为错误（如同在 *Visual Studio 2017* 中一样），并建议更新到 `preview` 或 `8.0`。

2. https://github.com/dotnet/roslyn/issues/38129 Visual Studio 2019 版本 16.3 错误地允许 `static` 局部函数调用非 `static` 局部函数。例如：

    ```c#
    void M()
    {
        void Local() {}

        static void StaticLocal()
        {
            Local();
        }
    }
    ```

    此类代码将在版本 16.4 中产生错误。

3. https://github.com/dotnet/roslyn/issues/35684 在 C# `7.1` 中，使用 `default` 字面量和未约束类型参数的二元运算符解析可能导致使用对象相等性，并为字面量提供 `object` 类型。
    例如，对于未约束类型 `T` 的变量 `t`，`t == default` 将被错误地允许并作为 `t == default(object)` 发出。
    在 *Visual Studio 2019 版本 16.4* 中，此场景现在将产生错误。

4. 在 C# `7.1` 中，`default as TClass` 和 `using (default)` 是被允许的。在 *Visual Studio 2019 版本 16.4* 中，这些场景现在将产生错误。

5. https://github.com/dotnet/roslyn/issues/38240 Visual Studio 2019 版本 16.3 错误地允许 `static` 局部函数使用目标需要捕获状态的委托创建表达式来创建委托。例如：

    ```c#
    void M()
    {
        object local;

        static void F()
        {
            _ = new Func<int>(local.GetHashCode);
        }
    }
    ```

    此类代码将在版本 16.4 中产生错误。

6. https://github.com/dotnet/roslyn/issues/37527 根据宿主体系结构的不同，编译器的常量折叠行为在将浮点常量转换为整型（如果不在 `unchecked` 上下文中则该转换会是编译时错误）时有所不同。现在在所有宿主体系结构上此类转换都产生零结果。

7. https://github.com/dotnet/roslyn/issues/38226 当 switch 表达式的多个分支之间存在公共类型，但有些分支包含没有类型的表达式（例如 `null`）无法转换为该公共类型时，编译器不当地将该公共类型推断为 switch 表达式的自然类型，从而导致错误。在 Visual Studio 2019 Update 4 中，修复了编译器，不再将此类 switch 表达式视为具有公共类型。

8. 用户定义的一元和二元运算符将从参数的可空性重新推断。这可能导致额外的警告：
    ```C#
    struct S<T>
    {
        public static S<T> operator~(S<T> s) { ... }
        public T F;
    }
    static S<T> Create<T>(T t) { ... }
    static void F()
    {
        object o = null;
        var s = ~Create(o);
        s.F.ToString(); // 警告: s.F 可能为 null
    }
    ```

9. https://github.com/dotnet/roslyn/issues/38469 在只允许类型的上下文中，在接口中查找名称时，编译器未在接口的基接口中查找该名称。现在在基接口中查找，并找到其中声明的类型（如果有与名称匹配的）。找到的类型可能与编译器以前找到的不同。

10. https://github.com/dotnet/roslyn/issues/38427 C# `7.0` 错误地允许具有元组名称差异的重复类型约束。在 *Visual Studio 2019 版本 16.4* 中，这是一个错误。
    ```C#
    class C<T> where T : I<(int a, int b)>, I<(int c, int d)> { } // 错误
    ```

11. 以前，未检查 `this ref` 和 `this in` 参数修饰符顺序的语言版本。在 *Visual Studio 2019 版本 16.4* 中，这些顺序在 langversion 低于 7.2 时会产生错误。参见 https://github.com/dotnet/roslyn/issues/38486

12. https://github.com/dotnet/roslyn/issues/40092 以前，编译器允许 `extern event` 声明具有初始化器，违反了 C# 语言规范。在 *Visual Studio 2019 版本 16.5* 中，此类声明会产生编译错误。
    ```C#
    class C
    {
        extern event System.Action E = null; // 错误
    }
    ```

13. https://github.com/dotnet/roslyn/issues/10492 在某些情况下，编译器会接受不遵守语言语法规则的表达式。例如 `e is {} + c`、`e is T t + c`。这些情况共同点是左操作数比 `+` 运算符松散，但左操作数不以表达式结尾，因此无法"消耗"加法。此类表达式在 Visual Studio 2019 版本 16.5 及更高版本中将不再被允许。

14. 在 Visual Studio 版本 15.0 及更高版本中，编译器会允许编译某些格式不正确的 `System.ValueTuple` 类型定义。在 *Visual Studio 2019 版本 16.5* 中，此类定义会产生警告。

15. 在 Visual Studio 版本 15.0 及更高版本中，编译器 API 会产生非泛型元组符号（`Arity` 为 0，没有类型参数，是原始定义）。在 *Visual Studio 2019 版本 16.5* 中，2-元组符号的 `Arity` 为 2，9-元组符号的 `Arity` 为 8（因为它是 `ValueTuple'8`）。

16. 在 *Visual Studio 2019 版本 16.5* 及 8.0 及更高版本的语言版本中，当没有类型 `System.Exception` 时，编译器将不再接受 `throw null`。

17. https://github.com/dotnet/roslyn/issues/39852 以前，编译器允许隐式索引或范围索引器的调用指定任何命名参数。在 *Visual Studio 2019 版本 16.5* 中，这些调用不再允许参数名称。

18. https://github.com/dotnet/roslyn/issues/36039 在 *Visual Studio 2019 版本 16.5* 中，编译器在成员体内也会尊重可空性流注解属性。

19. https://github.com/dotnet/roslyn/issues/36039 在 *Visual Studio 2019 版本 16.5* 中，在重写或实现中的可空性流注解属性（如 `[MaybeNull]` 或 `[NotNull]`）的使用将被检查以遵守 null 规范。例如：
``` csharp
public class Base<T>
{
    [return: NotNull]
    public virtual T M() { ... }
}
public class Derived : Base<string?>
{
    public override string? M() { ... } // Derived.M 不遵守 Base.M 通过其 [NotNull] 属性作出的可空性声明
}
```

20. 在 *Visual Studio 2019 版本 16.9* 及更高版本中，编译器将不再允许对约束类型参数使用查询语法。例如：

```csharp
using System;
using System.Collections.Generic;
 
class C
{
    static void M<T>() where T : C
    {
        var q = from x in T select x; // error CS0119: 'T' 是类型参数，在给定上下文中无效
    }

    static Func<Func<int, object>, IEnumerable<object>> Select = null;
}
```

21. https://github.com/dotnet/roslyn/issues/50182 在 *Visual Studio 2019 版本 16.9* 及更高版本中，编译器将不再允许对实现了格式错误的 `IAsyncEnumerable`（具有非可选的 `CancellationToken` 参数）的变量使用 `await foreach`。

22. https://github.com/dotnet/roslyn/issues/49596 在 *Visual Studio 2019 版本 16.9* 及更高版本中，从 `sbyte` 或 `short` 到 `nuint` 的转换需要显式转换，而带有 `sbyte` 或 `short` 与 `nuint` 参数的二元操作需要对一个或两个操作数进行显式转换。
