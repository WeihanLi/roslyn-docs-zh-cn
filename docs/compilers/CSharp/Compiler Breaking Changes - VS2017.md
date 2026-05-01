**本文档列出了 Roslyn 2.0（VS2017）相对于 Roslyn 1.\*（VS2015）和原生 C# 编译器（VS2013 及更早版本）中已知的重大更改。**

*重大更改按单调递增的编号列表格式化，以便通过简写方式引用（例如"已知重大更改 #1"）。
每个条目应包括对重大更改的简短描述，以及描述重大更改完整详情的链接或内联的完整详情。*

1. 以前，当类型仅在动态性方面有所不同并涉及某些嵌套时，我们无法在多个类型之间（用于三元表达式和方法类型推断）找到最佳类型。例如，我们无法在 `KeyValuePair<dynamic, object>` 和 `KeyValuePair<object, dynamic>` 之间找到最佳类型。从 VS 2017 开始，我们能够找到最佳类型（我们合并了动态性）。在此示例中，`KeyValuePair<dynamic, dynamic>` 是最佳类型。这可能影响重载解析，例如允许选择以前被丢弃的更好重载。参见 issues [#12585](https://github.com/dotnet/roslyn/issues/12585)、[#14247](https://github.com/dotnet/roslyn/issues/14247)、[#14213](https://github.com/dotnet/roslyn/issues/14213) 以及其相应的 PR 了解更多详情。
