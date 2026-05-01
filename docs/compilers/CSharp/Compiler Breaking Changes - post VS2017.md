# VS2017 之后 C# 的重大更改

## VS2017（C# 7）之后的更改

- https://github.com/dotnet/roslyn/issues/17089
在 C# 7 中，编译器接受形如 `dynamic identifier` 的模式，例如 `if (e is dynamic x)`。只有当表达式 `e` 的静态类型为 `dynamic` 时才被接受。编译器现在拒绝在模式变量声明中使用 `dynamic` 类型，因为没有任何对象的运行时类型是 `dynamic`。

- https://github.com/dotnet/roslyn/issues/17674 在 C# 7 中，编译器接受形如 `_ = M();` 的赋值语句，其中 M 是 `void` 方法。编译器现在拒绝这种写法。

- https://github.com/dotnet/roslyn/issues/17173 在 C# 7.1 之前，`csc` 会接受 `/langversion` 选项中的前导零。现在应该拒绝。例如：`csc.exe source.cs /langversion:07`。

- https://github.com/dotnet/csharplang/issues/415
在 C# 7.0 中，元组字面量中的元素可以显式命名，但在 C# 7.1 中，没有显式命名的元素将获得推断名称。这使用与匿名类型中未显式命名的成员相同的规则。
例如，`var t = (a, b.c, this.d);` 将生成元素名称为 "a"、"c" 和 "d" 的元组。因此，对元组成员的调用可能会得到与 C# 7.0 不同的结果。
考虑 `a` 的类型为 `System.Func<bool>` 的情况，写 `var local = t.a();`。现在将找到元组的第一个元素并调用它，而以前只能意味着"调用名为 'a' 的扩展方法"。

- https://github.com/dotnet/roslyn/issues/16870 在 C# 7.0 及 C# 7.1 之前，编译器接受解构赋值中的自赋值。编译器现在会为此产生警告。例如，在 `(x, y) = (x, 2);` 中。

- https://github.com/dotnet/roslyn/issues/19151 编译器现在在检测到错误的模式匹配操作时更加精确，因为表达式不可能匹配该模式。以下情况现在会导致错误：
  1. `bool M(int? i) => i is long l; // error CS8121: 'int?' 类型的表达式无法由 'long' 类型的模式处理。`
  2. 以及其他整数类型不同的情况
  3. 相同的错误可能发生在其他模式匹配上下文中（即 `switch`）

- https://github.com/dotnet/roslyn/issues/17963 在 C# 7.0 及 C# 7.1 之前，编译器在对可空元组类型使用 "as" 运算符时，会将元组元素名称差异和动态性差异视为重要因素。现在编译器生成正确的值，而不是 `null`，并且不再产生"始终为 null"的警告。

- https://github.com/dotnet/roslyn/issues/20208 在 C# 7.0 及 C# 7.2 之前，编译器会认为使用 `var` 类型和元组字面量值声明的局部变量是已使用的，因此如果该局部变量未被使用，不会报告警告。编译器现在会产生诊断。例如，`var unused = (1, 2);`。

- https://github.com/dotnet/roslyn/issues/20873 在 Roslyn 2.3 中，`EmitOptions` 构造函数的 `includePrivateMembers` 参数被更改为使用 `true` 作为默认值。这是一个二进制兼容性中断。因此，使用此 API 的客户端可能需要重新编译，以获取新的默认值。

- https://github.com/dotnet/roslyn/issues/21582 在 C# 7.1 中，当引入 `default` 字面量时，它被允许在 null 合并运算符的左侧使用。例如，在 `default ?? 1` 中。在 C# 7.2 中，这个编译器 bug 被修复以符合规范，改为产生错误（"运算符 '??' 不能应用于操作数 'default'"）。

- https://github.com/dotnet/roslyn/issues/21979 在 C# 7.1 及更早版本中，编译器允许将接收者类型为 `System.TypedReference` 的实例方法的方法组转换为委托类型。这样的代码在运行时会抛出 `System.InvalidProgramException`。在 C# 7.2 中，这是一个编译时错误。

    ``` c#
    static Func<int> M(__arglist)
    {
        ArgIterator ai = new ArgIterator(__arglist);
        while (ai.GetRemainingCount() > 0)
        {
            TypedReference tr = ai.GetNextArg();
            return tr.GetHashCode; // 委托转换会导致后续的 System.InvalidProgramException
        }

        return null;
    }
    ```

- https://github.com/dotnet/roslyn/issues/21485 在 Roslyn 2.0 中，`unsafe` 修饰符可以在局部函数上使用而无需使用 `/unsafe` 编译标志。在 Roslyn 2.6（Visual Studio 2017 版本 15.5）中，编译器需要 `/unsafe` 编译标志，如果未使用该标志则会产生诊断。

- https://github.com/dotnet/roslyn/issues/20210 在 C# 7.2 中，对于新的模式 switch 构造的某些用法（其中 switch 表达式是常量），编译器将产生以前未产生的警告或错误。

    ``` c#
    switch (default(object))
    {
      case bool _:
      case true:  // 新错误：case 被前面的 case 包含
      case false: // 新错误：case 被前面的 case 包含
        break;
    }

    switch (1)
    {
      case 1 when true:
        break;
      default:
        break; // 新警告：不可达代码
    }
    ```

- https://github.com/dotnet/roslyn/issues/20103 在 C# 7.2 中，当针对声明模式中的常量 null 表达式进行测试（其中类型不是推断的）时，编译器现在会警告该表达式永远不是所提供类型的实例。

    ``` c#
    const object o = null;
    if (o is object res) { // warning CS0184: 给定的表达式永远不是所提供（'object'）类型的实例
    ```

- https://github.com/dotnet/roslyn/issues/22578 在 C# 7.1 中，编译器会为使用 `default` 字面量声明的可空类型的可选参数计算错误的默认值。例如，`void M(int? x = default)` 将使用 `0` 作为默认参数值，而不是 `null`。在 C# 7.2（Visual Studio 2017 版本 15.5）中，此类情况会计算正确的默认参数值（`null`）。

- https://github.com/dotnet/roslyn/issues/21979 在 Visual Studio 2017 版本 15.6 中，将 ref-like 类型（如 `TypedReference`）的实例方法转换为委托的操作将被明确禁止，并导致编译时错误。
示例：`Func<int> f = default(TypedReference).GetHashCode; // 新错误 CS0123: 'GetHashCode' 没有与委托 'Func<int>' 匹配的重载`

- https://github.com/dotnet/roslyn/pull/23416 在 Visual Studio 2017 版本 15.6 之前，编译器接受带有 void 类型参数的 `__arglist(...)` 表达式。例如，`__arglist(Console.WriteLine())`。但这样的程序在运行时会失败。在 Visual Studio 2017 版本 15.6 中，这会导致编译时错误。

- https://github.com/dotnet/roslyn/pull/24023 在 Visual Studio 2017 版本 15.6 中，`Microsoft.CodeAnalysis.CSharp.Syntax.CrefParameterSyntax` 构造函数和 `Update()` 的参数 `refOrOutKeyword` 被重命名为 `refKindKeyword`（如果使用命名参数，这是源代码级重大更改）。

- Visual Studio 2017 版本 15.0-15.5 存在一个有关局部函数确定赋值的 bug，当未调用的局部函数中包含捕获变量的嵌套 lambda 时，不会产生确定赋值错误。在 15.6 中已修复，现在会产生变量未确定赋值的错误。

    ```csharp
    void Method()
    {
        void Local()
        {
            Action a = () =>
            {
                int x;
                x++; // 在 15.0 - 15.5 中无错误
            };
        }
    }
    ```

- Visual Studio 2017 版本 15.7：https://github.com/dotnet/roslyn/issues/19792 C# 编译器现在将拒绝应该具有 [InAttribute] modreq 但没有的 [IsReadOnly] 符号。

- Visual Studio 2017 版本 15.7：https://github.com/dotnet/roslyn/pull/25131 C# 编译器现在将检查 `stackalloc T [count]` 表达式，以查看 T 是否满足 `Span<T>` 的约束。

- https://github.com/dotnet/roslyn/issues/24806 在 C# 7.2 及更早版本中，在编译过程中可能会观察到 RValue 表达式被简化为可以通过引用传递的变量。C# 7.3 将保留 rvalue 的行为，其中值必须始终通过副本传递。

    ```C#
    static int x = 123;
    static string Test1()
    {
        // 不能用对 "x" 的引用替换 "x + 0" 的值
        // 因为那会让方法看到 M1() 中的变化
        return (x + 0).ToString(M1());
    }

    static string M1()
    {
        x = 42;
        return "";
    }
    ```

- Visual Studio 2017 版本 15.7：https://github.com/dotnet/roslyn/issues/25450 对 `default` 表达式的使用新增了限制：
  - `e is default` 将产生新错误
  - `case default:` 将产生新错误

- Visual Studio 2017 版本 15.7：https://github.com/dotnet/roslyn/issues/25399 如果分部方法的实现与定义中的 ref 类型不同，C# 编译器现在将产生错误。

- Visual Studio 2017 版本 15.7：https://github.com/dotnet/roslyn/issues/23525 如果向嵌入的 pdb 提供了无效的 pdbpath，C# 编译器现在将产生错误，而不只是将其写入二进制文件。

- Visual Studio 2017 版本 15.7：元组相等功能（在 C# 7.3 中）引入了元组类型的内置 `==` 和 `!=` 运算符。这些内置运算符优先于自定义 `ValueTuple` 类型上的用户定义比较运算符。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/22455 如果 `__arglist` 调用中有 "in" 或 "out" 参数，C# 编译器现在将产生错误。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/pull/27882 如果将局部变量 ref 赋值给具有更宽逃逸范围的参数（且为 ref-like 类型），C# 编译器现在将产生错误。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/26418 C# 编译器现在将对具有 "ref" 或 "ref readonly" ref 类型的 out 变量声明产生错误。示例：`M(out ref int x);`

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/27047 C# 编译器现在将在将标记为已过时的运算符用作元组比较的一部分时产生诊断。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/pull/27461 `LanguageVersionFacts.TryParse` 方法不再是扩展方法。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/pull/27803 模式匹配现在在尝试将栈绑定的值返回到无效的逃逸范围时将产生错误。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/26743 如果 `Bar` 是包含类型的固定字段，C# 现在将拒绝如 `public int* M => &this.Bar[0];` 这样的表达式。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/27772 调用接收者现在将在嵌套范围表达式中检查逃逸范围错误。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/28117 C# 现在将拒绝对 byval 参数的 ref 赋值。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/issues/27049 C# 现在将拒绝受限类型中的 `base.Method()` 调用，因为这需要装箱。
