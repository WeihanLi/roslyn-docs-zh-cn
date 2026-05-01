## 本文档列出了 .NET 5 之后 Roslyn 中已知的重大更改。

1. https://github.com/dotnet/roslyn/issues/46044 在 C# 9.0（Visual Studio 16.9）中，当将 `default` 赋值给或将可能为 `null` 的值转型为未被约束为值类型或引用类型的类型参数时，会报告警告。要避免此警告，可以用 `?` 对类型进行注解。
    ```C#
    static void F1<T>(object? obj)
    {
        T t1 = default; // warning CS8600: 将可能为 null 的值转换为不可为 null 的类型。
        t1 = (T)obj;    // warning CS8600: 将可能为 null 的值转换为不可为 null 的类型。

        T? t2 = default; // 正常
        t2 = (T?)obj;    // 正常
    }
    ```

2. https://github.com/dotnet/roslyn/pull/50755 在 .NET 5.0.200（Visual Studio 16.9）中，如果条件表达式的两个分支之间存在公共类型，则该类型是条件表达式的类型。

   这是 5.0.103（Visual Studio 16.8）的重大更改，该版本由于 bug 错误地使用条件表达式的目标类型作为类型，即使两个分支之间存在公共类型。

   这一最新更改使编译器行为与 C# 规范以及 .NET 5.0 之前的编译器版本保持一致。
    ```C#
    static short F1(bool b)
    {
        // 16.7, 16.9          : CS0266: 无法将 'int' 隐式转换为 'short'
        // 16.8                : 正常
        // 16.8 -langversion:8 : CS8400: 功能"目标类型条件表达式"在 C# 8.0 中不可用
        return b ? 1 : 2;
    }

    static object F2(bool b, short? a)
    {
        // 16.7, 16.9          : int
        // 16.8                : short
        // 16.8 -langversion:8 : CS8400: 功能"目标类型条件表达式"在 C# 8.0 中不可用
        return a ?? (b ? 1 : 2);
    }
    ```

3. https://github.com/dotnet/roslyn/issues/52630 在 C# 9（.NET 5，Visual Studio 16.9）中，record 可能会使用基类型中的隐藏成员作为位置成员。在 Visual Studio 16.10 中，这现在是一个错误：
```csharp
record Base
{
    public int I { get; init; }
}
record Derived(int I) // 与此参数对应的位置成员 'Base.I' 被隐藏。
    : Base
{
    public int I() { return 0; }
}
```

4. 在 .NET 5 和 Visual Studio 16.9（及更早版本）中，顶级语句可以在包含名为 `Program` 的类型的程序中使用。在 .NET 6 和 Visual Studio 17.0 中，顶级语句生成 `Program` 类的部分声明，因此任何用户定义的 `Program` 类型也必须是部分类。

```csharp
System.Console.Write("top-level");
Method();

partial class Program
{
    static void Method()
    {
    }
}
```

5. https://github.com/dotnet/roslyn/issues/53021 C# 现在会对显式接口实现中错误放置的 ```::``` token 报告错误。以下示例代码中：

    ``` C#
    void N::I::M()
    {
    }
    ```

    以前的 Roslyn 版本不会报告任何错误。

    现在会对 M 之前的 ```::``` token 报告错误。
