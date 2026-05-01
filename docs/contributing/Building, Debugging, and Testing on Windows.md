# 在 Windows 上构建、调试和测试

## 使用代码

使用命令行，可以使用以下模式开发 Roslyn：

1. 克隆 https://github.com/dotnet/roslyn
1. 运行 Restore.cmd
1. 运行 Build.cmd
1. 运行 Test.cmd

## 推荐的 .NET Framework 版本

所需的最低 .NET Framework 版本是 4.7.2。

## 使用 Visual Studio 2022 开发

1. 使用最新的 [Visual Studio 2022 预览版](https://visualstudio.microsoft.com/vs/preview/vs2022/)
    - 确保在选定的工作负载中包含 Visual Studio 扩展开发
    - 确保在选定的单个组件中包含 C# 和 Visual Basic、MSBuild 和 .NET Core
    - 确保在工具 -> 选项 -> 环境 -> 预览功能中勾选"使用 .NET Core SDK 预览版"
    - 重启 Visual Studio
1. 安装与 [global.json](https://github.com/dotnet/roslyn/blob/main/global.json#L3) 中的 `sdk.version` 属性匹配的 [.NET 10.0 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)
1. [PowerShell 5.0 或更新版本](https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-windows-powershell)。如果您使用的是 Windows 10，则没问题；只有在使用更早版本的 Windows 时才需要升级。下载链接在["升级现有 Windows PowerShell"](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-windows-powershell?view=powershell-6#upgrading-existing-windows-powershell)标题下。
1. 运行 Restore.cmd
1. 打开 Roslyn.slnx

## 使用 Visual Studio Code 开发

请参阅[在 Unix 上构建、调试和测试](Building,%20Debugging,%20and%20Testing%20on%20Unix.md#working-in-visual-studio-code)文档以开始使用 Visual Studio Code 开发 Roslyn。

## 运行测试

有多种选项可以运行核心 Roslyn 单元测试：

### 命令行

Test.cmd 脚本将在已构建的二进制文件上运行单元测试。可以传递 `-build` 参数以在运行测试前强制进行新构建。

1. 从开始菜单运行"VS2022 开发人员命令提示符"。
2. 导航到您的 Git 克隆目录。
3. 在命令提示符中运行 `Test.cmd`。

您可以通过直接使用相关选项运行 eng/build.ps1 脚本来更精确地控制测试的运行方式。例如，传入 `-test` 开关将在 .NET Framework 上运行测试，而传入 `-testCoreClr` 开关将在 .NET Core 上运行测试。

测试结果可以在 artifacts/TestResults 目录中查看。

### 测试资源管理器

可以从测试资源管理器窗口运行和调试测试。为了获得最佳性能，我们建议以下操作：

1. 打开**工具 &rarr; 选项... &rarr; 测试**
    1. 勾选**从源文件实时发现测试**复选框
    2. 取消勾选**另外从生成程序集发现测试...**复选框
2. 使用测试资源管理器的搜索框将可见测试的范围缩小到您正在处理的功能
3. 当您没有主动运行测试时，将搜索查询设置为 `__NonExistent__` 以从 UI 隐藏所有测试

#### 使用 WSL 在 Linux 中测试

可以在 WSL 下运行和调试测试。这首次需要一些设置。之后，只需在测试资源管理器中选择 WSL 作为活动环境即可。

1. 通过运行 `wsl --install` 安装 WSL（[详细信息](https://docs.microsoft.com/en-us/windows/wsl/setup/environment#get-started)）
2. 在 VS 安装程序中，将".NET Debugging with WSL"安装为单个组件
3. 在测试资源管理器中，进入"配置远程测试环境"（在齿轮图标下）并取消注释 wsl/Ubuntu 环境
4. 从测试资源管理器下拉列表中选择该测试环境
5. 从测试资源管理器测试列表运行测试。首次这样做时会提示您将一些 dotnet 组件安装到 WSL 环境中。
6. 调试测试。首次这样做时会提示您将一些远程调试组件安装到 WSL 环境中。

[更多详细信息](https://docs.microsoft.com/en-us/visualstudio/debugger/debug-dotnet-core-in-wsl-2)

### WPF 测试运行器

要调试测试，您可以右键单击包含测试的测试项目，然后选择**设为启动项目**。然后按 F5。这将在命令行运行器下运行测试。团队的一些成员一直在开发一个 GUI 运行器，允许选择单个测试等。从 [xunit.runner.wpf](https://github.com/pilchie/xunit.runner.wpf) 获取源代码，构建并尝试一下。

## 在 Visual Studio 中尝试您的更改

### 使用命令行部署（推荐方法）

您可以使用以下命令构建和部署：
`.\Build.cmd -Restore -Configuration Release -deployExtensions -launch`。

然后您可以使用 `devenv /rootSuffix RoslynDev` 启动 `RoslynDev` 蜂巢。

### 使用 F5 部署

Rosyln 解决方案旨在支持通过 F5 轻松调试。我们的几个项目生成 VSIX，在构建期间部署到 Visual Studio 中。F5 操作将使用这些覆盖我们安装的二进制文件的 VSIX 启动一个新的 Visual Studio 实例。这意味着尝试对语言、IDE 或调试器的更改就像按 F5 一样简单。请注意，对于编译器的更改，进程外构建不会使用私有构建的编译器版本。

启动项目需要设置为 **RoslynDeployment**。这应该是默认设置，但在某些情况下需要显式设置。要设置它，请在解决方案资源管理器中右键单击 RoslynDeployment 项目，然后选择"设为启动项目"。

**RoslynDeployment** 是位于 `src/Deployment` 文件夹中的容器项目，它将所有主要的 Roslyn 扩展捆绑并部署在一起。当您将 RoslynDeployment 设置为启动项目并按 F5 时，它将一次部署所有以下扩展，为您提供包含所有 Roslyn 组件的完整调试体验。

如果您正在处理特定区域并希望通过仅部署相关扩展来优化构建时间，您可以改为将以下单个项目之一设置为启动项目：

- **Roslyn.VisualStudio.Setup**：此项目可以在解决方案资源管理器的 VisualStudio\Setup 文件夹中找到，并构建 Roslyn.VisualStudio.Setup.vsix。它包含提供 C# 和 VB 编辑的核心语言服务。它还包含用于驱动 Visual Studio 中 IntelliSense 和语义分析的编译器副本。虽然这是用于生成波浪线和其他信息的编译器副本，但它不是您进行构建时用于实际生成最终 .exe 或 .dll 的编译器。如果您正在修复 IDE 错误，这是您要使用的项目。
- **Roslyn.Compilers.Extension**：此项目可以在解决方案资源管理器的 Compilers\Extension 文件夹中找到，并构建 Roslyn.Compilers.Extension.vsix。这部署了用于在 IDE 中进行实际构建的命令行编译器副本。它只影响从安装到的 Visual Studio 实验实例触发的构建，因此不会影响您的常规构建。请注意，如果您只安装此项，IDE 将不知道您构建中包含的任何语言功能。如果您经常处理新的语言功能，您可能希望考虑同时构建 CompilerExtension 和 VisualStudioSetup 项目，以确保实际构建和实时分析同步。
- **ExpressionEvaluatorPackage**：此项目可以在解决方案资源管理器的 ExpressionEvaluator\Package 文件夹中找到，并构建 ExpressionEvaluatorPackage.vsix。这部署了表达式评估器和结果提供程序，这些组件由调试器用于在监视窗口、即时窗口等中解析和评估 C# 和 VB 表达式。这些组件仅在调试时使用。

Roslyn 使用的实验实例是一个完全独立的 Visual Studio 实例，具有自己的设置和已安装的扩展。默认情况下，它也是与其他 Visual Studio SDK 项目使用的标准"实验实例"不同的实例。如果您熟悉 Visual Studio 蜂巢的概念，我们部署到 RoslynDev 根后缀中。

### 使用 VSIX 和 NuGet 包部署

如果您想在日常使用 Visual Studio 时尝试您的扩展，您可以在 Binaries 文件夹中找到您构建的带有 .vsix 扩展名的扩展。您可以双击扩展将其安装到您的主 Visual Studio 蜂巢中。这将替换基本安装版本。安装后，您会在工具 > 扩展和更新中看到它标记为"实验性"，表示您正在运行实验版本。您可以通过选择您的版本并单击卸载来卸载您的版本并返回到最初安装的版本。

如果您只安装 VSIX，则 IDE 将正常运行（即新的编译器和 IDE 行为），但构建操作或从命令行构建不会。要解决此问题，请将您构建的 `Microsoft.Net.Compilers.Toolset` 的引用添加到您的 csproj 中。如下所示，您需要（1）添加指向本地构建文件夹的 nuget 源，（2）添加包引用，然后（3）使用程序中包含的 `#error version` 验证项目的构建输出。

## 在 Visual Studio Code 中尝试您的更改

请参阅 [VSCode 文档](https://github.com/dotnet/vscode-csharp/blob/main/CONTRIBUTING.md#using-a-locally-developed-roslyn-server)。

## 各种其他提示

### 引用引导编译器

如果您对 Roslyn 编译器进行了更改并想用它构建任何项目，您可以使用安装了 **CompilerExtension** 的 Visual Studio 蜂巢，或者从命令行使用 `/p:BootstrapBuildPath=YourBootstrapBuildPath` 运行 msbuild。`YourBootstrapBuildPath` 可以是您机器上的任何目录，只要它内部有 csc 和 vbc。您可以检查 cibuild.cmd 看看它是如何使用的。

### 排查您的设置

要确认正在使用哪个版本的编译器，请在您的程序中包含 `#error version`，编译器将产生一个诊断，包括其自己的版本以及它正在运行的语言版本。

您还可以将调试器附加到 Visual Studio 并检查已加载的模块，查看各种 `CodeAnalysis` 模块加载的文件夹（`RoslynDev` 应该从 `AppData` 下的某个位置加载它们，而不是从 `Program File`）。

### 在 [dotnet/runtime](https://github.com/dotnet/runtime) 仓库上测试

1. 确保您可以将 `runtime` 仓库作为基线构建（运行 `build.cmd libs+libs.tests`，这应该足以构建所有 C# 代码，如果提示则安装任何先决条件，可能首先需要 `git clean -xdf` 和 `build.cmd -restore` - 有关特定先决条件和构建说明，请参阅 [runtime 仓库文档](https://github.com/dotnet/runtime/blob/main/docs/workflow/README.md)）
2. 在您的 `roslyn` 仓库上运行 `build.cmd -pack -c Release`
    - 请注意，也可以使用 `-c Debug`（同时将下面 `RestoreAdditionalProjectSources` 属性值中的 `Release` 更改为 `Debug`）。这将允许在构建运行时检查编译器的调试断言。
    - 对于我们来说，调查编译运行时库导致编译器的调试断言失败的场景是很好的。但是，断言失败可能会掩盖编译器最终是否成功构建运行时并生成正确的二进制文件。因此，如果目标只是检查"功能性破坏"，那么首先使用 Release 模式编译器可能更可取。
3. 在您的 NuGet 包缓存中找到编译器工具集（其位置可能是 `%NUGET_PACKAGES%\microsoft.net.compilers.toolset` 或 `%userprofile%\.nuget\packages\microsoft.net.compilers.toolset`）。
4. 如果 NuGet 缓存中的工具集版本与您刚刚在 Roslyn 中打包的工具集版本相同，请从缓存中删除该工具集版本。这允许 NuGet 获取您刚刚在本地创建的新包，而不是使用过时的缓存版本。
5. 类似于[此提交](https://github.com/RikkiGibson/runtime/commit/runtime-local-roslyn-build-example)修改您的本地 `runtime` 注册，然后再次构建
    - 使用到您的 `roslyn` 仓库的本地路径将 `<RestoreAdditionalProjectSources><PATH-TO-YOUR-ROSLYN-ENLISTMENT>\artifacts\packages\Release\Shipping\</RestoreAdditionalProjectSources>` 添加到 `Directory.Build.props`
    - 使用您刚刚打包的包版本（查看上述 artifacts 文件夹）将 `<UsingToolMicrosoftNetCompilers>true</UsingToolMicrosoftNetCompilers>` 和 `<MicrosoftNetCompilersToolsetVersion>5.3.0-dev</MicrosoftNetCompilersToolsetVersion>` 添加到 `eng/Versions.props`

### 测试 VS 插入

请参阅[此处](https://microsoft.sharepoint.com/teams/managedlanguages/_layouts/OneNote.aspx?id=%2Fteams%2Fmanagedlanguages%2Ffiles%2FTeam%20Notebook%2FRoslyn%20Team%20Notebook%20-%20New&wd=target%28Infrastructure.one%7C0E3C75FE-0827-41BF-9696-F5CD7105BC9A%2FRun%20RPS%20against%20Roslyn%20PR%7C7A74A1E4-1507-41BC-84F1-F9159C853177%2F%29)的内部文档了解该过程。

### 使用额外的 IOperation 验证进行测试

运行 `build.cmd -testIOperation`，它将 `ROSLYN_TEST_IOPERATION` 环境变量设置为 `true` 并运行测试。
要在 IDE 中运行这些测试，最简单的方法是找到 `//#define ROSLYN_TEST_IOPERATION` 指令并取消注释。
请参阅 [IOperation 测试钩子](https://github.com/dotnet/roslyn/blob/main/docs/compilers/IOperation%20Test%20Hook.md)文档了解更多详细信息。

### 复制 Used Assemblies 分支中的失败

为了复制该分支中的测试失败，有几个选项：

1. 取消注释 `src/Compilers/Test/Core/Compilation/CompilationExtensions.cs:9`，它定义了 `ROSLYN_TEST_USEDASSEMBLIES`，并运行您的测试。请_不要_将此签入，因为它将为每个项目中的每个测试启用测试钩子，并显著减慢常规测试运行速度。
2. 设置 `ROSLYN_TEST_USEDASSEMBLIES` 环境变量并在设置后重启 VS。
3. 在 `src/Compilers/Test/Utilities/CSharp/CSharpTestBase.cs` 中的 `CSharpTestBase.VerifyUsedAssemblyReferences` 开头设置断点。当它中断时，使用 VS 的跳转到位置或将指令指针拖过 `EnableVerifyUsedAssemblyReferences` 上的早期检查并返回。

当测试失败被隔离时，请为此添加一个_专用_测试（即即使未启用 Used Assemblies 验证也会失败），以便更容易避免将来的回归。
最好不要复制整个原始测试，只需足够触发错误以确保它受到回归保护。
在将相关修复推送到 CI 之前，您可以使用 `build.cmd` 的 `-testUsedAssemblies` 命令行选项在本地验证。例如：`build.cmd -testCoreClr -testCompilerOnly -testUsedAssemblies`。

### 运行 PublicAPI 修复程序

1. 恢复并构建 `Compilers.slnf`。这是确保源生成器项目已构建并且我们可以在运行 `dotnet format` 时加载生成器程序集所必需的。
   ```ps1
   C:\Source\roslyn> .\restore.cmd
   C:\Source\roslyn> .\build.cmd
   ```
2. 调用 `dotnet format`（.NET SDK 中包含的那个，而不是已弃用的全局工具 `dotnet-format`）。只运行分析器子命令并只修复"缺少 Public API 签名"诊断。我们还必须传递 `--include-generated` 标志以在分析中包含源生成的文档。
   ```ps1
   C:\Source\roslyn> cd ..
   C:\Source> dotnet format analyzers .\roslyn\Compilers.slnf --diagnostics=RS0016 --no-restore --include-generated -v diag
   ```

## 贡献

有关将更改贡献回代码的详细信息，请参阅[贡献代码](https://github.com/dotnet/roslyn/blob/main/CONTRIBUTING.md)。
