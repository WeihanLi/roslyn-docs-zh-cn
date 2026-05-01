编译器工具集 NuPkg
===
## 概述
编译器从 Roslyn 的所有主要分支生成 [Microsoft.Net.Compilers.Toolset NuPkg](https://www.nuget.org/packages/Microsoft.Net.Compilers.Toolset)。安装此 NuPkg 后，它将使用对应分支构建的版本覆盖 MSBuild 自带的编译器。

此包旨在支持以下场景：
1. 允许编译器团队为遇到阻塞性问题的客户提供快速热修复。客户可以安装此包，直到修复版本随 .NET SDK 或 Visual Studio 服务更新正式发布。
1. 作为 Roslyn 二进制文件在更大范围的 .NET SDK 构建流程中的传输机制。
1. 允许客户在各种 Roslyn 构建版本上进行实验，其中许多版本尚未包含在正式发布的产品中。

此包**不**旨在支持在旧版 MSBuild 中使用更新的编译器版本。例如，在 MSBuild 15 中使用 Microsoft.Net.Compilers.Toolset 3.5（C# 8）是明确不支持的场景。

希望将编译器作为受支持构建基础设施一部分的客户，应使用 [Visual Studio Build Tools SKU](https://docs.microsoft.com/en-us/visualstudio/install/workload-component-id-vs-build-tools?view=vs-2022]) 或 [.NET SDK](https://dotnet.microsoft.com/download/visual-studio-sdks)。

## NuPkg 安装

运行以下命令安装 NuPkg：

```cmd
> nuget install Microsoft.Net.Compilers.Toolset   # 安装 C# 和 VB 编译器
```

项目的每日 NuGet 构建版本也可在我们的 [Azure DevOps feed](https://dev.azure.com/dnceng/public/_packaging?_a=feed&feed=dotnet-tools) 中获取：

> https://pkgs.dev.azure.com/dnceng/public/_packaging/dotnet-tools/nuget/v3/index.json

## Microsoft.Net.Compilers

[Microsoft.Net.Compilers](https://www.nuget.org/packages/Microsoft.Net.Compilers) NuPkg 已被弃用。它是 Microsoft.Net.Compilers.Toolset 的 .NET Desktop 专用版本，在 3.6.0 发布之后将不再生产。对于所有受支持场景，Microsoft.Net.Compilers.Toolset 包是其直接替代品。
