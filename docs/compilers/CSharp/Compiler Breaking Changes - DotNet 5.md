## 本文档列出了 Roslyn 在 C# 9.0 中已知的重大更改，这些更改将随 .NET 5（Visual Studio 2019 版本 16.8）引入。

1. 从 C# 9.0 开始，当你对 `byte` 或 `sbyte` 类型的值进行 switch 操作时，编译器会跟踪哪些值已被处理、哪些尚未处理。从技术上讲，我们对所有数值类型都会如此，但实际上只有 `byte` 和 `sbyte` 类型会产生重大更改。例如，以下程序包含一个 switch 语句，显式处理了开关控制表达式所有可能的值：
    ```csharp
    int M(byte b)
    {
        switch (b)
        {
            case 0: case 1: case 2: ...
                return 0;
        }
        return 1;
    }
    ```
    由于该方法对输入的每个值都会执行 return，因此后续语句（`return 1;`）被认为不可达，会产生不可达代码警告。之前，编译器不分析值集合的完整性，后续语句被认为是可达的。该代码在 C# 8.0 中编译时没有警告，但在 C# 9.0 中会产生警告。

    类似地，对 `byte` 或 `sbyte` 类型输入的所有值都显式处理的 switch 表达式被认为是完整的。例如，以下程序

    ```csharp
    int M(byte b)
    {
        return b switch
        {
            0 => 0, 1 => 0, 2 => 0, ... byte.MaxValue => 0,
        };
    }
    ```

    在 C# 8.0 中编译会产生警告（switch 表达式未处理其输入的所有值），但在 C# 9.0 中不会产生警告。反之，向其中添加回退分支会在 C# 8.0 中消除警告，但在 C# 9.0 中会导致错误（该情况已被之前的 case 处理）：

    ```csharp
    int M(byte b)
    {
        return b switch
        {
            0 => 0, 1 => 0, 2 => 0, ... byte.MaxValue => 0,
            _ => 1 // C# 9.0 中的错误
        };
    }
    ```

2. 在 C# 9.0 中，模式中不允许使用 `not` 作为类型
    在 C# 9.0 中，我们引入了一个 `not` 模式，用于对后续模式取反：
    ```csharp
        bool IsNull(object o) => o is not null;
    ```
    我们建议使用 `not null` 模式作为检查值是否非 null 的最清晰方式。

    由于 C# 9.0 中的模式可以以 `not` 作为模式的一部分开头，因此以 `not` 开头的模式不再被视为引用名为 `not` 的类型。表达式 `o is not x` 以前用于声明类型为 `not` 的变量 `x`。现在，它检查输入 `o` 是否与名为 `x` 的常量不同。

3. 在 C# 9.0 中，`and` 和 `or` 不允许用作模式指示符
    在 C# 9.0 中，我们引入了 `and` 和 `or` 模式组合子来组合其他模式：
    ```csharp
       bool IsSmall(int i) => o is 0 or 1 or 2;
    ```
    我们还引入了类型模式，使你无需使用标识符来命名正在声明的变量：
    ```csharp
        bool IsSignedIntegral(object o) =>
            o is sbyte or short or int or long;
    ```
    由于 `and` 和 `or` 组合子可以跟在类型模式之后，编译器将其解释为模式组合子的一部分，而不是声明模式的标识符。因此，从 C# 9.0 开始，使用 `or` 或 `and` 作为模式变量标识符是错误的。

4. https://github.com/dotnet/roslyn/pull/44841 在 *C# 9* 及更高版本中，编译器将 `record` 标识符（既可作为类型语法，也可作为 record 声明）的歧义解析为选择 record 声明。以下示例现在将被视为 record 声明：

    ```C#
    abstract class C
    {
        record R2() { }
        abstract record R3();
    }
    ```

5. 对 https://github.com/dotnet/roslyn/issues/44067 的修复生成了正确的（不同的）代码。
   在某些情况下，编译器曾生成行为根据 CLR 规范具有歧义的代码。编译器以前会在这些情况下产生警告 CS1957。
   编译器现在生成正确的无歧义代码，而不是报告 CS1957。由于代码生成策略的更改可能会改变程序的运行时行为，这可能是一个重大更改。如果你的程序以前没有触发警告 CS1957，则这不影响你的代码。
