

## 常规信息
* [编译器和语言特性状态](https://github.com/dotnet/roslyn/blob/main/docs/Language%20Feature%20Status.md)
* [破坏性变更日志](https://github.com/dotnet/roslyn/blob/main/docs/compilers/CSharp/Compiler%20Breaking%20Changes%20-%20post%20VS2017.md)
* [NuGet 包](https://github.com/dotnet/roslyn/blob/main/docs/wiki/NuGet-packages.md)
* [C# 语言版本历史](https://github.com/dotnet/csharplang/blob/main/Language-Version-History.md)

## Visual Studio 2017 版本 15.7

C# 编译器现在支持 7.3 语言特性集，包括：
- `System.Enum`、`System.Delegate` 和 [`unmanaged`](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/blittable.md) 约束。
- [Ref 局部变量重新赋值](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/ref-local-reassignment.md)：ref 局部变量和 ref 参数现在可以使用 ref 赋值运算符（`= ref`）重新赋值。
- [stackalloc 初始化器](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/stackalloc-array-initializers.md)：栈分配的数组现在可以初始化，例如 `Span<int> x = stackalloc[] { 1, 2, 3 };`。
- [可移动 fixed 缓冲区的索引](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/indexing-movable-fixed-fields.md)：fixed 缓冲区现在无需先固定即可进行索引。
- [自定义 `fixed` 语句](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/pattern-based-fixed.md)：实现了适当 `GetPinnableReference` 的类型可以在 `fixed` 语句中使用。
- [改进的重载候选](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/improved-overload-candidates.md)：某些重载解析候选可以提前排除，从而减少歧义。
- [初始化器和查询中的表达式变量](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/expression-variables-in-initializers.md)：`out var` 等表达式变量和模式变量现在可以在字段初始化器、构造函数初始化器和 LINQ 查询中使用。
- [元组比较](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/tuple-equality.md)：元组现在可以使用 `==` 和 `!=` 进行比较。
- [支持字段上的特性](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.3/auto-prop-field-attrs.md)：允许在自动实现属性上使用 `[field: …]` 特性以针对其支持字段。


## Visual Studio 2017 版本 15.6

C# 编译器现在支持：
* CoreCLR 上的编译器服务器，用于提升构建吞吐量性能
* CoreCLR 上的强名称签名（[`/keyfile` 选项](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/keyfile-compiler-option)，支持所有操作系统）

对 7.2 语言特性进行了两处细微语言变更：
* `in` 重载的优先级决定（[详情](https://github.com/dotnet/csharplang/issues/945)）
* 放宽 ref 扩展方法中 `ref` 和 `this` 的顺序要求（[详情](https://github.com/dotnet/csharplang/issues/1022)）

[已发布 API](TODO)，[Bug 修复](https://github.com/dotnet/roslyn/pulls?q=is%3Apr+milestone%3A15.6+is%3Aclosed)
 
## [Visual Studio 2017 版本 15.5](https://github.com/dotnet/roslyn/releases/tag/Visual-Studio-2017-Version-15.5)

C# 编译器现在支持 7.2 语言特性集，包括：

* 支持通过 `ref struct` 修饰符在整个 Kestrel 和 CoreFX 中使用 `Span<T>` 类型。
* `readonly struct` 修饰符：强制要求结构体的所有成员均为 `readonly`。这为代码增加了一层正确性保障，并使编译器能够在访问成员时避免不必要的值复制。
* `in` 参数 / `ref readonly` 返回：允许以与可修改的 `ref` 值相同的效率安全地传递和返回不可修改的结构体。
* `private protected` 访问修饰符：将访问限制为 `protected` 和 `internal` 的交集。
* 非尾随命名参数：命名参数现在可以在参数列表中间使用，而不要求其后所有参数也必须按名称传递。

[Bug 修复](https://github.com/dotnet/roslyn/pulls?q=is%3Apr+milestone%3A15.5+is%3Aclosed)
 
## [Visual Studio 2017 版本 15.3](https://github.com/dotnet/roslyn/releases/tag/Visual-Studio-2017-Version-15.3)

C# 编译器现在支持 7.1 语言特性集，包括：
- [异步 Main 方法](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.1/async-main.md)
- ["default" 字面量](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.1/target-typed-default.md)
- [推断的元组元素名称](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.1/infer-tuple-names.md)
- [泛型的模式匹配](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.1/generics-pattern-match.md)

C# 和 VB 编译器现在可以生成[引用程序集](https://github.com/dotnet/roslyn/blob/main/docs/features/refout.md)。

当你在项目中使用 C# 7.1 特性时，灯泡会提示将项目的语言版本升级为"C# 7.1"或"latest"。

[已发布 API](https://github.com/dotnet/roslyn/commit/5520eaccd5d22ae98a39a5f88120277f02097dbf)，[Bug 修复](https://github.com/dotnet/roslyn/pulls?q=is%3Apr+milestone%3A15.3+is%3Aclosed)
 
## [Visual Studio 2017 版本 15.0](https://github.com/dotnet/roslyn/releases/tag/Visual-Studio-2017)
 C# 编译器现在支持 [7.0](https://blogs.msdn.microsoft.com/dotnet/2017/03/09/new-features-in-c-7-0/) 语言特性集，包括：
- [out 变量](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/out-var.md)
- [模式匹配](https://github.com/dotnet/csharplang/blob/main/proposals/patterns.md)
- [元组](https://github.com/dotnet/roslyn/blob/main/docs/features/tuples.md)
- [解构](https://github.com/dotnet/roslyn/blob/main/docs/features/deconstruction.md)
- [弃元](https://github.com/dotnet/roslyn/blob/main/docs/features/discards.md)
- [局部函数](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/local-functions.md)
- [二进制字面量](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/binary-literals.md)
- [数字分隔符](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/digit-separators.md)
- ref 返回和局部变量
- [泛化的异步返回类型](https://github.com/dotnet/roslyn/blob/main/docs/features/task-types.md)
- 更多表达式体成员
- [throw 表达式](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/throw-expression.md)
