# IDE CodeStyle 分析器文档

## 概述

1. 所有 [IDE 分析器诊断 ID](../../src/Analyzers/Core/Analyzers/IDEDiagnosticIds.cs) 的官方文档已添加到 `dotnet/docs` 仓库，位于 <https://github.com/dotnet/docs/tree/main/docs/fundamentals/code-analysis/style-rules>。

2. 每个 IDE 诊断 ID 都有专属的文档页面。例如：
   1. 带有代码样式选项的诊断：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0011>
   2. 不带代码样式选项的诊断：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0004>
   3. 共享相同选项的多个诊断 ID：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0003-ide0009>。在这种情况下，必须为所有诊断 ID 添加[重定向](https://github.com/dotnet/docs/blob/7ae7272d643aa1c6db96cad8d09d4c2332855960/.openpublishing.redirection.fundamentals.json#L11-L14)。

3. 我们提供了 IDE 诊断 ID 和代码样式选项的表格索引，以便于参考和导航：
   1. 主索引（规则 ID + 代码样式选项）：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/>
   2. 次要索引（仅代码样式选项）：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/language-rules#net-style-rules>
   3. 每个首选项/选项组的索引。例如：
      1. 修饰符首选项：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/modifier-preferences>
      2. 表达式级别首选项：<https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/expression-level-preferences>

## 添加或更新 IDE 分析器时的操作项

Roslyn IDE 团队（@dotnet/roslyn-ide）负责确保 IDE 分析器的文档保持最新。每当我们添加新的 IDE 分析器、添加新的代码样式选项或更新现有 IDE 分析器的语义时，需要完成以下工作：

1. **在 [dotnet/docs](https://github.com/dotnet/docs) 仓库中需要采取的行动**：
   1. 如果你在_为 IDE 分析器创建 Roslyn PR_：
      1. 请按照[为"IDExxxx"规则贡献文档](https://docs.microsoft.com/contribute/dotnet/dotnet-contribute-code-analysis#contribute-docs-for-idexxxx-rules)中的步骤，将 IDE 分析器的文档添加到 [dotnet/docs](https://github.com/dotnet/docs) 仓库。
      2. 理想情况下，文档 PR 应与 Roslyn PR 同步创建，或在 Roslyn PR 获得批准/合并后立即创建。
   2. 如果你在_审查 IDE 分析器的 Roslyn PR_：
      1. 请确保分析器作者知晓上述文档贡献要求，尤其是社区 PR。
      2. 如果文档 PR 未与 Roslyn PR 同步创建，请确保在 `dotnet/docs` 仓库中提交一个[新的文档问题](https://github.com/dotnet/docs/issues)来跟踪此工作。

2. **Roslyn 仓库中无需采取行动**：帮助链接已在基础分析器类型中与 Roslyn 仓库中的 IDE 代码样式分析器关联，因此每个新的 IDE 分析器诊断 ID 将自动获得 `https://docs.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ideXXXX` 作为其帮助链接。Roslyn PR 上无需任何操作来确保此链接。

如果你认为现有 IDE 规则文档信息不足或不正确，请向 [dotnet/docs](https://github.com/dotnet/docs) 仓库提交 PR 以更新文档。
