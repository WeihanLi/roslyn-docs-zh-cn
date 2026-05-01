编译器对 AnalyzerConfig 的支持
===================================

AnalyzerConfig 是一种 EditorConfig 超集（https://editorconfig.org/）文件格式，由 Roslyn 命令行编译器识别。分析器配置文件中指定的选项以两种方式被编译器识别：遵循 `dotnet_diagnostic.<diagnostic-id>.severity = <value>` 模式的选项键由编译器解析并解释，用于配置编译器诊断的严重性。`<diagnostic-id>` 代表编译器匹配的诊断 ID（不区分大小写）。`<value>` 必须是 [ReportDiagnostic](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/Portable/Diagnostic/ReportDiagnostic.cs) 枚举中某个成员的名称（同样不区分大小写）。这些设置随后会按语法树逐一应用到路径与编译中 AnalyzerConfig 名称规范匹配的每个文件上。

不符合上述模式的属性将被视为分析器选项，并放置在 [AnalyzerOptions 类型](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/Portable/DiagnosticAnalyzer/AnalyzerOptions.cs) 的 `PerTreeOptionsProvider` 中，供分析器使用。

AnalyzerConfig 文件可以通过 `/analyzerconfig:<file-path>` 参数传递给命令行编译器。
