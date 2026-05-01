## 本文档列出了 Roslyn 在 C# 10.0 中已知的重大更改，这些更改将随 .NET 6 引入。

1. <a name="1"></a>从 C# 10.0 开始，模式中不再允许使用 null 抑制运算符。

    ```csharp
    void M(object o)
    {
        if (o is null!) {} // 错误
    }
    ```

2. <a name="2"></a>在 C# 10 中，具有推断类型的 lambda 表达式和方法组可隐式转换为 `System.MulticastDelegate`，以及 `System.MulticastDelegate` 的基类和接口（包括 `object`）；
lambda 表达式和方法组也可隐式转换为 `System.Linq.Expressions.Expression` 和 `System.Linq.Expressions.LambdaExpression`。
这些是*函数类型转换*。

    新的隐式转换可能会在编译器迭代搜索重载并在第一个包含任何适用重载的类型或命名空间范围处停止的情况下，更改重载解析结果。

    a. 实例方法和扩展方法

    ```csharp
    class C
    {
        static void Main()
        {
            var c = new C();
            c.M(Main);      // C#9: E.M(); C#10: C.M()
            c.M(() => { }); // C#9: E.M(); C#10: C.M()
        }
    
        void M(System.Delegate d) { }
    }

    static class E
    {
        public static void M(this object x, System.Action y) { }
    }
    ```

    b. 基类和派生类方法

    ```csharp
    using System;
    using System.Linq.Expressions;

    class A
    {
        public void M(Func<int> f) { }
        public object this[Func<int> f] => null;
        public static A operator+(A a, Func<int> f) => a;
    }

    class B : A
    {
        public void M(Expression e) { }
        public object this[Delegate d] => null;
        public static B operator+(B b, Delegate d) => b;
    }

    class Program
    {
        static int F() => 1;

        static void Main()
        {
            var b = new B();
            b.M(() => 1);   // C#9: A.M(); C#10: B.M()
            _ = b[() => 2]; // C#9: A.this[]; C#10: B.this[]
            _ = b + F;      // C#9: A.operator+(); C#10: B.operator+()
        }
    }
    ```

    c. 方法组或匿名方法转换为 `Expression` 或 `LambdaExpression`

    ```csharp
    using System;
    using System.Linq.Expressions;

    var c = new C();
    c.M(F);                         // error CS0428: 无法将方法组 'F' 转换为非委托类型 'Expression'
    c.M(delegate () { return 1; }); // error CS1946: 匿名方法表达式无法转换为表达式树

    static int F() => 0;

    class C
    {
        public void M(Expression e) { }
    }

    static class E
    {
        public static void M(this object o, Func<int> a) { }
    }
    ```

3. <a name="3"></a>在 C# 10 中，具有推断类型的 lambda 表达式可以贡献影响重载解析的参数类型。

    ```csharp
    using System;

    class Program
    {
        static void F(Func<Func<object>> f, int i) { }
        static void F(Func<Func<int>> f, object o) { }

        static void Main()
        {
            F(() => () => 1, 2); // C#9: F(Func<Func<object>>, int); C#10: 歧义
        }
    }
    ```

4. <a name="4"></a><a name="roslyn-58339"></a>在 Visual Studio 17.0.7 中，如果带有主构造函数的 `record struct` 中的显式构造函数有一个调用隐式无参构造函数的 `this()` 初始化器，则会报告错误。参见 [roslyn#58339](https://github.com/dotnet/roslyn/pull/58339)。

    例如，以下代码会产生错误：
    ```csharp
    record struct R(int X, int Y)
    {
        // error CS8982: 在带有参数列表的 'record struct' 中声明的构造函数必须具有调用主构造函数或显式声明的构造函数的 'this' 初始化器。
        public R(int x) : this() { X = x; Y = 0; }
    }
    ```

    可以通过在 `this()` 初始化器中调用主构造函数（如下所示），或通过声明一个调用主构造函数的无参构造函数来解决此错误。
    ```csharp
    record struct R(int X, int Y)
    {
        public R(int x) : this(x, 0) { } // 正常
    }
    ```

5. <a name="5"></a><a name="roslyn-57925"></a>在 Visual Studio 17.0.7 中，如果没有构造函数的 `struct` 类型声明包含某些但不是所有字段的初始化器，编译器将报告所有字段必须被赋值的错误。

    17.0 的早期版本跳过了编译器在此场景中合成的无参构造函数的*确定赋值分析*，没有报告未赋值的字段，可能导致实例包含未初始化的字段。更新后的分析和错误报告与显式声明的构造函数一致。参见 [roslyn#57925](https://github.com/dotnet/roslyn/pull/57925)。

    例如，以下代码会产生错误：
    ```csharp
    struct S // error CS0171: 在控制返回给调用方之前，必须完全赋值字段 'S.Y'
    {
        int X = 1;
        int Y;
    }
    ```

    要解决错误，请声明一个无参构造函数并为没有初始化器的字段赋值，或者删除现有字段初始化器以使编译器不再合成无参构造函数。
    （为了与 Visual Studio 17.1 的兼容性，该版本要求带有字段初始化器的 `struct` [包含显式声明的构造函数](https://github.com/dotnet/roslyn/blob/main/docs/compilers/CSharp/Compiler%20Breaking%20Changes%20-%20DotNet%207.md#6)，请避免在不声明构造函数的情况下向其余字段添加初始化器。）

    例如，上述示例中的错误可以通过添加构造函数并为 `Y` 赋值来解决：
    ```csharp
    struct S
    {
        int X = 1;
        int Y;
        public S() { Y = 0; } // 正常
    }
    ```
