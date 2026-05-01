**本文档列出了 Roslyn 3.0（Visual Studio 2019）相对于 Roslyn 2.\*（Visual Studio 2017）中已知的重大更改**

*重大更改按单调递增的编号列表格式化，以便通过简写方式引用（例如"已知重大更改 #1"）。
每个条目应包括对重大更改的简短描述，以及描述重大更改完整详情的链接或内联的完整详情。*

1. https://github.com/dotnet/roslyn/issues/27800 在左侧操作数为 dynamic 的复合赋值加法/减法表达式中，C# 现在将保留从左到右的求值顺序。以下示例代码中：

    ``` C#
    class DynamicTest
    {
        public int Property { get; set; }
        static dynamic GetDynamic() => return new DynamicTest();
        static int GetInt() => 1;
        public static void Main() => GetDynamic().Property += GetInt();
    }
    ```

    以前版本的 Roslyn 的求值顺序为：
    1. GetInt()
    2. GetDynamic()
    3. get_Property
    4. set_Property

    现在的求值顺序为：
    1. GetDynamic()
    2. get_Property
    3. GetInt()
    4. set_Property

2. 以前，允许添加在其中声明了 `Microsoft.CodeAnalysis.EmbeddedAttribute` 或 `System.Runtime.CompilerServices.NonNullTypesAttribute` 类型的模块。
    在 Visual Studio 2019 中，这会与这些类型的注入声明产生冲突错误。

3. 以前，可以引用在被引用程序集中声明的 `System.Runtime.CompilerServices.NonNullTypesAttribute` 类型。
    在 Visual Studio 2019 中，来自程序集的类型会被忽略，优先使用该类型的注入声明。

4. https://github.com/dotnet/roslyn/issues/29656 以前，返回 ref 的异步局部函数可以编译，通过忽略返回类型的 `ref` 修饰符。
    在 Visual Studio 2019 中，这现在会产生错误，就像返回 ref 的异步方法一样。

5. https://github.com/dotnet/roslyn/issues/27748 C# 现在会对 `e is _` 形式的表达式产生警告。*is_type_expression* 可以与名为 `_` 的类型一起使用。但是，在 C# 8 中，我们引入了写作 `_` 的丢弃模式（不能用作 *is_pattern_expression* 的*模式*）。为了减少混淆，当你写 `e is _` 时会出现新警告：

    ``` none
    (11,31): warning CS8413: 名称 '_' 引用类型 'Program1._'，而不是丢弃模式。请使用 '@_' 表示类型，或使用 'var _' 丢弃。
        bool M1(object o) => o is _;
    ```

6. C# 8 在 `switch` 语句使用名为 `_` 的常量作为 case 标签时会产生警告。

    ``` none
    (1,18): warning CS8512: 名称 '_' 引用常量 '_'，而不是丢弃模式。请使用 'var _' 丢弃该值，或使用 '@_' 按名称引用常量。
    switch(e) { case _: break; }
    ```

7. 在 C# 8.0 中，当被切换的表达式是元组表达式时，switch 语句的括号是可选的，因为元组表达式有自己的括号：

    ``` c#
    switch (a, b)
    ```

    由于这一变化，`SwitchStatementSyntax` 节点的 `OpenParenToken` 和 `CloseParenToken` 字段现在有时可能为空。

8. 在 *is-pattern-expression* 中，当常量表达式因其值而与提供的模式不匹配时，现在会发出警告。此类代码以前被接受但没有警告。例如：

    ``` c#
    if (3 is 4) // 警告：给定表达式永远不匹配提供的模式。
    ```

    当常量表达式*始终*与 *is-pattern-expression* 中的常量模式匹配时，我们也会发出警告。例如：

    ``` c#
    if (3 is 3) // 警告：给定表达式始终匹配提供的常量。
    ```

    模式始终匹配的其他情况（例如 `e is var t`）不会触发警告，即使编译器知道它们会产生不变的结果。

9. https://github.com/dotnet/roslyn/issues/26098 在 C# 8 中，当输入类型是开放类类型而被测试的类型是值类型时，如果 is-type 表达式始终为 `false`，会产生警告：

    ``` c#
    class C<T> { }
    void M<T>(C<T> x)
    {
        if (x is int) { } // 警告：给定表达式永远不是所提供的 ('int') 类型。
    }
    ```

    以前，我们只在反向情况下给出警告：

    ``` c#
    void M<T>(int x)
    {
        if (x is C<T>) { } // 警告：给定表达式永远不是所提供的 ('C<T>') 类型。
    }
    ```

10. 以前，引用程序集的生成包含嵌入资源。在 Visual Studio 2019 中，嵌入资源不再被生成到引用程序集中。
  参见 https://github.com/dotnet/roslyn/issues/31197

11. ref struct 现在通过模式支持释放。具有可访问的 `void Dispose()` 实例方法的 ref struct 枚举器现在将在枚举结束时调用它，无论该 struct 类型是否实现了 IDisposable：

    ``` c#
    public class C
    {
        public ref struct RefEnumerator
        {
            public int Current => 0;
            public bool MoveNext() => false;
            public void Dispose() => Console.WriteLine("仅在 C# 8.0 中调用");
        }

        public RefEnumerator GetEnumerator() => new RefEnumerator();

        public static void Main()
        {
            foreach(var x in new C())
            {
            }
            // 在 C# 8.0 中，RefEnumerator.Dispose() 将在此处被调用
        }
    }
    ```

12. https://github.com/dotnet/roslyn/issues/32732 同时处理 `true` case 和 `false` case 的 switch 语句现在被认为处理了所有可能的输入值。
