# roslyn-docs-cn

## 简介

Roslyn 中文文档，原英文文档参考 <https://github.com/dotnet/roslyn/tree/master/docs>

## 欢迎来到 .NET 编译器平台（`Roslyn`）

Roslyn 提供了开源的有着丰富的代码分析api的 C# 和 Visual Basic 编译器，它使用Visual Studio使用的相同API来启用构建代码分析工具。

### 语言设计讨论

我们现在正在别的项目中进行语言功能的讨论：

- C# 相关的问题 <https://github.com/dotnet/csharplang>
- VB 相关的问题 <https://github.com/dotnet/vblang>
- 对两种语言都有影响的功能 <https://github.com/dotnet/csharplang >

### 下载 C# 和 VisualBasic

想使用 C# 和 VisualBaisc 开始开发？ 下载已经内置了最新的功能特性的 [Visual Studio 2017](https://www.visualstudio.com/downloads/)。在Visual Studio 2017 中也已经安装了预构建的Azure虚拟机映像。

要在不使用Visual Studio的情况下安装最新版本，请运行以下nuget命令行之一：

``` bash
nuget install Microsoft.Net.Compilers   # Install C# and VB compilers
nuget install Microsoft.CodeAnalysis    # Install Language APIs and Services
```

Daily NuGet builds of the project are also available in our MyGet feed.

> <https://dotnet.myget.org/F/roslyn/api/v3/index.json>

### 源代码

- 克隆源代码: `git clone https://github.com/dotnet/roslyn.git`
- 基于 Roslyn 构建的 [增强的源代码视图](http://source.roslyn.io/)
- [使用源代码进行开发、测试和调试](https://github.com/dotnet/roslyn/wiki/Building%20Testing%20and%20Debugging)

### Get Started

- MSDN Magazine 上 Alex Turner 写的教程文章
  - [使用 Roslyn 写一个动态的代码分析器](https://msdn.microsoft.com/en-us/magazine/dn879356)
  - [为你的 Roslyn 增加一个代码修复功能](https://msdn.microsoft.com/en-us/magazine/dn904670.aspx)

- [Roslyn 概览](https://github.com/dotnet/roslyn/wiki/Roslyn%20Overview)
- [API 变化从 CTP6 到 RC](https://github.com/dotnet/roslyn/wiki/VS-2015-RC-API-Changes)
- [示例和实践](https://github.com/dotnet/roslyn/wiki/Samples-and-Walkthroughs)
- [文档](https://github.com/dotnet/roslyn/tree/master/docs)
- [分析器文档](https://github.com/dotnet/roslyn/tree/master/docs/analyzers)
- [语法可视化工具](https://github.com/dotnet/roslyn/wiki/Syntax%20Visualizer)
- [语法标注工具](http://roslynquoter.azurewebsites.net/)
- [路线图](https://github.com/dotnet/roslyn/wiki/Roadmap)
- [语言设计笔记](https://github.com/dotnet/roslyn/issues?q=label%3A%22Design+Notes%22+)
- [FAQ](https://github.com/dotnet/roslyn/wiki/FAQ)
- 不要忘记看一下我们的 [Wiki](https://github.com/dotnet/roslyn/wiki) 以了解更多信息

## Contact

contact me at <weihanli@outlook.com>