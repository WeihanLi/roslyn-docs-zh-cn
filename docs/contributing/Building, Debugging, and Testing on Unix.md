# 在 Unix 上构建、调试和测试
本指南旨在帮助开发人员设置从 Linux 调试/贡献 Roslyn 的环境。
特别是对于没有 Linux 上 .NET Core 开发经验的开发人员。

## 使用代码
1. 确保命令 `git` 和 `curl` 可用
1. 克隆 git@github.com:dotnet/roslyn.git
1. 运行 `./build.sh --restore`
1. 运行 `./build.sh --build`

## 在 Visual Studio Code 中工作
1. 安装 [VS Code](https://code.visualstudio.com/Download)
    - 安装 VS Code 后，安装 [C# 扩展](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)
    - 重要提示：您可以通过按 *Ctrl+Shift+P* 或按 *Ctrl+P* 并输入 `>` 字符来按名称查找编辑器命令。这将帮助您熟悉下面提到的编辑器命令。在 Mac 上，使用 *⌘* 代替 *Ctrl*。
1. 安装与 [global.json](../../global.json#L3) 中的 `sdk.version` 属性匹配的 [.NET 10.0 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)
3. 您可以通过运行 *Run Build Task* 命令从 VS Code 构建，然后选择适当的任务，如 *build* 或 *build current project*（后者为您在编辑器中查看的当前文件构建包含项目）。
4. 您可以通过在编辑器中打开测试类，然后使用 *Run Tests in Context* 和 *Debug Tests in Context* 编辑器命令从 VS Code 运行测试。您可能想将这些命令绑定到与其 Visual Studio 等效项匹配的键盘快捷键（**Ctrl+R, T** 用于 *Run Tests in Context*，**Ctrl+R, Ctrl+T** 用于 *Debug Tests in Context*）。
5. 您可以通过运行 "launch vscode with language server" 任务从当前代码启动带有语言服务器的新 VS Code 实例。

## 运行测试
单元测试可以通过运行 `./build.sh --test` 执行。

要在单个项目中运行所有测试，建议使用 `dotnet test path/to/project` 命令。

## GitHub
克隆和推送的最佳方式是使用 SSH。在 Windows 上，您通常使用 HTTPS，这与双因素身份验证不直接兼容（需要 PAT）。SSH 设置更简单，GitHub 有一个很棒的 HOWTO 来设置。

https://help.github.com/articles/connecting-to-github-with-ssh/

## 调试测试失败
调试的最佳方式是使用带有 SOS 插件的 lldb。这与 WinDbg 中使用的 SOS 相同，如果您熟悉它，lldb 调试将非常简单。

[dotnet/diagnostics](https://github.com/dotnet/diagnostics) 仓库有更多信息：

- [获取 LLDB](https://github.com/dotnet/diagnostics/blob/main/documentation/lldb/linux-instructions.md)
- [安装 SOS](https://github.com/dotnet/diagnostics/blob/main/documentation/installing-sos-instructions.md)
- [使用 SOS](https://github.com/dotnet/diagnostics/blob/main/documentation/sos-debugging-extension.md)

CoreCLR 也有一些针对特定 Linux 调试场景的指南：

- https://github.com/dotnet/coreclr/blob/main/Documentation/botr/xplat-minidump-generation.md
- https://github.com/dotnet/coreclr/blob/main/Documentation/building/debugging-instructions.md#debugging-core-dumps-with-lldb

更正：
- LLDB 和 createdump 必须以 root 身份运行
- `dotnet tool install -g dotnet-symbol` 必须从 `$HOME` 运行

### 核心转储
CoreClr 不使用 Linux 上的标准核心转储机制。相反，您必须通过环境变量指定您希望生成核心转储。最简单的设置是执行以下操作：

```
> export COMPlus_DbgEnableMiniDump=1
> export COMPlus_DbgMiniDumpType=4
```

这将导致生成完整内存转储，然后可以加载到 LLDB 中。

[dotnet-dump](https://github.com/dotnet/diagnostics/blob/main/documentation/dotnet-dump-instructions.md) 预览版也可用于交互式创建和分析转储。

### GC 压力失败
当您怀疑存在与测试相关的 GC 失败时，可以使用以下环境变量来帮助追踪。

```
> export COMPlus_HeapVerify=1
> export COMPlus_gcConcurrent=1
```

`COMPlus_HeapVerify` 变量导致 GC 在每次进入和退出时运行验证例程。将崩溃并为 GC 团队提供更可操作的跟踪。

`COMPlus_gcConcurrent` 变量删除 GC 中的并发性。这有助于隔离这是 GC 失败还是 GC 之外的内存损坏。这应该在您使用 `COMPLUS_HeapVerify` 确定确实在 GC 中崩溃后设置。

注意：这些变量也可以在 Windows 上使用。

## Ubuntu 18.04
开发 Roslyn 的推荐操作系统是 Ubuntu 18.04。本指南是使用 Ubuntu 18.04 编写的，但应该适用于大多数 Linux 环境。这里选择 Ubuntu 18.04 是因为它支持 Hyper-V 中的增强 VM。这使得从 Windows 机器上使用更容易：全屏、复制/粘贴等...

### Hyper-V
Hyper-V 有一个内置的 Ubuntu 18.04 镜像，支持增强模式。有关更多信息，请参阅[创建此类镜像的教程](https://docs.microsoft.com/virtualization/hyper-v-on-windows/quick-start/quick-create-virtual-machine)。

按照此操作时，请确保：

1. 单击安装源并取消选中"Windows 安全启动"
1. 完成 Ubuntu 安装向导。在完成之前，全屏模式将不可用。

总体而言，这大约需要 5-10 分钟才能完成。

### 源链接
许多需要构建的仓库使用源链接，由于依赖项更改，它在 Ubuntu 18.04 上崩溃。
要禁用源链接，请将以下内容添加到仓库根目录中的 `Directory.Build.props` 文件中。

``` xml
<EnableSourceControlManagerQueries>false</EnableSourceControlManagerQueries>
<EnableSourceLink>false</EnableSourceLink>
<DeterministicSourcePaths>false</DeterministicSourcePaths>
```
### 先决条件

确保通过 `apt install` 安装以下内容

- clang
- lldb
- cmake
- xrdp
