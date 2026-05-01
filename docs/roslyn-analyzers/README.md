# Roslyn 分析器

## Microsoft.CodeAnalysis.Analyzers

*最新稳定版本：* <sub>[![NuGet](https://img.shields.io/nuget/v/Microsoft.CodeAnalysis.Analyzers.svg)](https://www.nuget.org/packages/Microsoft.CodeAnalysis.Analyzers)</sub>

*最新预发布版本：* [此处](https://dev.azure.com/dnceng/public/_artifacts/feed/dotnet7/NuGet/Microsoft.CodeAnalysis.Analyzers/versions)

此包包含用于正确使用 [Microsoft.CodeAnalysis](https://www.nuget.org/packages/Microsoft.CodeAnalysis) NuGet 包（即 .NET 编译器平台"Roslyn"）API 的规则。这些规则主要面向诊断分析器和代码修复提供程序的作者，帮助其以推荐方式调用 Microsoft.CodeAnalysis API。[更多关于此包中规则的信息](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/Microsoft.CodeAnalysis.Analyzers/Microsoft.CodeAnalysis.Analyzers.md)

## Roslyn.Diagnostics.Analyzers

*最新稳定版本：* <sub>[![NuGet](https://img.shields.io/nuget/v/Roslyn.Diagnostics.Analyzers.svg)](https://www.nuget.org/packages/Roslyn.Diagnostics.Analyzers)</sub>

*最新预发布版本：* [此处](https://dev.azure.com/dnceng/public/_artifacts/feed/dotnet7/NuGet/Roslyn.Diagnostics.Analyzers/versions)

此包包含专门针对 .NET 编译器平台（"Roslyn"）项目（即 [dotnet/roslyn](https://github.com/dotnet/roslyn) 仓库）的规则。此分析器包**不面向 Roslyn 仓库以外的一般用途**。[更多关于此包中规则的信息](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/Roslyn.Diagnostics.Analyzers/Roslyn.Diagnostics.Analyzers.md)

## Microsoft.CodeAnalysis.BannedApiAnalyzers

*最新稳定版本：* <sub>[![NuGet](https://img.shields.io/nuget/v/Microsoft.CodeAnalysis.BannedApiAnalyzers.svg)](https://www.nuget.org/packages/Microsoft.CodeAnalysis.BannedApiAnalyzers)</sub>

*最新预发布版本：* [此处](https://dev.azure.com/dnceng/public/_artifacts/feed/dotnet7/NuGet/Microsoft.CodeAnalysis.BannedApiAnalyzers/versions)

此包包含可自定义的规则，用于识别对被禁止 API 的引用。[更多关于此包中规则的信息](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/Microsoft.CodeAnalysis.BannedApiAnalyzers/Microsoft.CodeAnalysis.BannedApiAnalyzers.md)

有关此分析器的使用说明，请参阅[使用指南](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/Microsoft.CodeAnalysis.BannedApiAnalyzers/BannedApiAnalyzers.Help.md)。

## Microsoft.CodeAnalysis.PublicApiAnalyzers

*最新稳定版本：* <sub>[![NuGet](https://img.shields.io/nuget/v/Microsoft.CodeAnalysis.PublicApiAnalyzers.svg)](https://www.nuget.org/packages/Microsoft.CodeAnalysis.PublicApiAnalyzers)</sub>

*最新预发布版本：* [此处](https://dev.azure.com/dnceng/public/_artifacts/feed/dotnet7/NuGet/Microsoft.CodeAnalysis.PublicApiAnalyzers/versions)

此包包含帮助库作者监控公开 API 变更的规则。[更多关于此包中规则的信息](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/PublicApiAnalyzers/Microsoft.CodeAnalysis.PublicApiAnalyzers.md)

有关此分析器的使用说明，请参阅[使用指南](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/PublicApiAnalyzers/PublicApiAnalyzers.Help.md)。
