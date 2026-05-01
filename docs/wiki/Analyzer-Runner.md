Analyzer Runner
-------------

AnalyzerRunner 目前作为 Roslyn 源代码的一部分存在。

## 运行步骤：

1. 从 dotnet/roslyn 检出最新的 main 分支
2. 运行 Restore.cmd 以恢复 NuGet 包
3. 在 Release 配置下构建 Roslyn.slnx，可在 Visual Studio 中完成，也可运行 Build.cmd -release
4. 将 AnalyzerRunner 设为启动项目
5. 选择启动配置（launchProperties.json 中指定了若干配置，也可以新建或修改现有配置）
6. 使用 Ctrl+F5 运行 AnalyzerRunner，或在命令行中运行
