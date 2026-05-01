# MSBuild 使用

## 支持不同的 MSBuild

此仓库必须支持使用多种不同的 MSBuild 配置进行构建：

- 通过 Visual Studio 的 MSBuild：当开发人员在 Visual Studio 中打开 Roslyn.slnx 并执行构建操作时会发生这种情况。这使用桌面 MSBuild 来驱动解决方案。
- 通过 CLI 的 MSBuild：仓库的跨平台部分通过 CLI 构建。这是 Roslyn.slnx 中包含的代码的子集。
- MSBuild xcopy：一个可 xcopy 的 MSBuild 版本，用于运行我们的许多 Jenkins 分支。它允许 Roslyn 在完全全新的 Windows 映像上构建和运行测试（无先决条件）。[xcopy-msbuild](https://github.com/jaredpar/xcopy-msbuild) 项目负责构建此映像。
- BuildTools：这是由 [dotnet/buildtools](https://github.com/dotnet/buildtools) 生成的工具集合，用于构建许多 dotnet 仓库。

这对我们的仓库提出了一个小的负担，需要保持我们的构建 props/targets 文件简单以避免任何奇怪的冲突。这在目前很少成为问题。

## 选择 MSBuild

鉴于我们的仓库支持多个 MSBuild 版本，在构建、还原等时必须选择一个使用。优先级列表如下：

1. 开发人员命令提示符：从开发人员命令提示符内调用时，将使用关联的 MSBuild。
1. 机器 MSBuild：在安装了 MSBuild 15.0 的机器上调用时，将使用第一个提到的实例。
1. XCopy MSBuild：当没有其他选项可用时的后备方案
