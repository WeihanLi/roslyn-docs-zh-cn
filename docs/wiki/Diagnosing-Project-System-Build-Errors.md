当你在 Visual Studio 中打开 C# 或 VB.NET 项目时，Visual Studio 需要解析你的项目文件以了解
编译器设置和引用。在大多数情况下，这很简单，但如果你手动自定义了项目文件，
或者使用了额外的 SDK 或 NuGet 包，有时可能会出错。本文将帮助你
调试出错的原因。

## 如何判断本指南是否适用于我？

有几种判断方式：

1. Microsoft 工程师要求你按照这些步骤操作。

2. 在任何版本的 Visual Studio 中，查看文本文件上方的第一个项目下拉框是否显示"Miscellaneous Files"，而不是
   你期望看到的项目名称：

    ![导航栏中显示 Miscellaneous Files](images/design-time-build-errors/miscellaneous-files.png)

3. 如果你使用的是 Visual Studio 2015 Update 2 或更高版本，请在错误列表中查找警告 IDE0006：
    ![IDE0006 错误示例](images/design-time-build-errors/ide0006.png)

## 如何获取日志文件以诊断 Visual Studio 2026 中发生的问题？

1. 打开"视图"菜单，选择"其他窗口"，然后选择"命令窗口"。
2. 命令窗口打开后，在其中输入 `Project.LogRoslynWorkspaceStructure`：

    ![命令窗口的示例截图](images/design-time-build-errors//typing-in-command-window.png)

3. 按 Enter。系统将弹出提示，要求你保存一个 XML 文件。此过程可能需要一些时间。如果你有问题报告，请将此文件私密地附加到问题报告中。

## 如何获取日志文件以诊断 Visual Studio 2017、2019 或 2022 中发生的问题？

1. 从 Visual Studio Marketplace 安装 Project System Tools 扩展：
   - [适用于 Visual Studio 2022 的版本](https://marketplace.visualstudio.com/items?itemName=VisualStudioProductTeam.ProjectSystemTools2022)
   - [适用于 Visual Studio 2017 和 2019 的版本](https://marketplace.visualstudio.com/items?itemName=VisualStudioProductTeam.ProjectSystemTools)
2. 在安装扩展时重启 Visual Studio。
3. 再次关闭 Visual Studio，在磁盘上找到你的解决方案文件，并删除与解决方案并排的 .vs 隐藏文件夹。如果看不到，你需要显示隐藏文件。
4. 打开 Visual Studio。暂时不要打开你的解决方案。
5. 依次选择"视图">"其他窗口">"Build Logging"。
6. 在 Build Logging 窗口中，点击开始按钮。
7. 打开你的解决方案，并打开出问题的文件。你应该会看到 Build Logging 窗口中出现各种条目。
8. 打开出问题的文件后，截取 Visual Studio 的截图。保存此截图并将其附加到你的反馈项目中。
9. 点击 Build Logging 窗口中的停止按钮。
10. 点击一个日志条目后按 Ctrl+A 全选所有日志。
11. 右键单击，选择"保存日志"。保存并将其附加到你的反馈项目中。
12. 打开"工具"菜单，选择"Log Roslyn Workspace Structure"。系统将提示保存 XML 文件，此过程可能需要一些时间。将此文件附加到你的反馈项目中。如果文件较大，且扩展没有为你创建 .zip 文件，你可能需要将其压缩为 .zip 文件。

## 如何获取日志文件以诊断 Visual Studio 2015 中发生的问题？

1. 关闭 Visual Studio。
2. 在磁盘上找到你的解决方案文件，并删除与解决方案并排的 .vs 隐藏文件夹。
3. 运行"开发者命令提示符"。最简单的方法是在开始菜单中搜索：

    ![运行开发者命令提示符](images/design-time-build-errors/run-developer-command-prompt.png)

4. 在此提示符中输入 `set TRACEDESIGNTIME=true`。
5. 输入 `devenv` 以从此提示符重新运行 Visual Studio。
6. 打开你的解决方案。

现在，打开你的 `%TEMP%` 目录，路径类似于 `C:\Users\<username>\AppData\Local\Temp`。在这里
你会看到一堆以 `.designtime.log` 结尾的文件。

- 如果你运行的是 Visual Studio 2015 Update 2 或更高版本，文件名以解决方案中某个项目的名称开头，
  然后是下划线，再是 Visual Studio 调用的底层构建目标名称。查找名为
  `YourProject_Compile_#####.designtime.log` 的文件，其中 ##### 只是为保持
  日志文件唯一性而生成的随机标识符。查找名称中含有"Compile"的日志文件。

- 如果你运行的是 Visual Studio 2015 Update 1 或更早版本，情况会稍微复杂一些。这些文件仅以随机
  GUID 命名，但仍以 .designtime.log 结尾。如果你打开日志文件，会首先看到一个显示变量的部分，
  然后是如下所示的一行：

  `Project "c:\Projects\ConsoleApplication53\ConsoleApplication53\ConsoleApplication53.csproj" (Compile target(s)):`

  这显示了项目的完整名称以及正在运行的目标（Compile）。同样，查找"Compile"目标。
  有些文件包含其他被调用的目标，你需要忽略那些文件。你可能需要考虑
  使用 Visual Studio 的"在文件中查找"功能来找到正确的文件。

找到项目对应的正确日志文件后，滚动到最末尾并确认是否存在错误。你应该
看到类似以下的内容：

    Build FAILED.

    c:\ConsoleApplication53\ConsoleApplication53\ConsoleApplication53.csproj(17,5): error : An error occurred!
        0 Warning(s)
        1 Error(s)

值得注意的是，你应该看到 `Build FAILED`，然后是一个或多个错误。这是日志中错误的摘要，所以如果
你确实看到了错误，现在应该在日志文件中搜索该错误并找到其所在位置。希望该错误能给出某种提示；
例如找不到某个文件或 SDK，或者某些权限被拒绝等。
在这种情况下，你可以联系相关负责人以找出出错的原因。

如果问题似乎是 Visual Studio 本身的问题，你可能需要在
[Roslyn GitHub 项目](https://github.com/dotnet/roslyn)上提交一个 bug，让 Visual Studio 工程师查看。请确保
提供完整的日志文件，如果可能的话还要提供你的项目文件，因为我们可能需要两者来诊断问题。
