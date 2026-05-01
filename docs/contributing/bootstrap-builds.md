# 引导构建

构建正确性测试确保 dotnet/roslyn 中的最新代码可以用来构建 dotnet/roslyn。本质上是我们的编译器可以自我引导。

## 为什么？

我们花费如此大的精力是因为这类问题在事后非常难以追踪。编译器是一个复杂的工具，dotnet/roslyn 是一个大型代码库。找出上次成功引导以来约 300 次提交中是哪一次导致当前引导失败，可能需要大量时间。

过去我们大约每月运行一次引导练习。每次使用最新编译器构建 dotnet/roslyn 时失败，通常需要数周时间才能了解是哪个更改导致了失败。这导致了**大量**时间浪费，几乎使我们多次错过发布时间。

构建正确性作业的引入就是为了解决这个问题。编译器在每次更改时都验证它可以引导自身，这意味着我们能立即发现失败。

## 流程

构建正确性作业首先构建 Microsoft.Net.Compilers.Toolset 包。这给我们一个包含最新更改的功能性编译器。此构建使用 `/define:BOOTSTRAP` 进行，允许编译器使失败更具可操作性。这主要体现在以下几个方面：

- 将 [ExitingTraceListener](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Shared/ExitingTraceListener.cs) 插入进程跟踪监听器。这意味着任何 `Debug.Assert` 失败都将导致编译失败，并显示可操作的堆栈跟踪。
- 定义 [ValidateBootstrap](https://github.com/dotnet/roslyn/blob/main/src/Compilers/Core/MSBuildTask/ValidateBootstrap.cs)。这让我们能够验证引导构建中使用的编译器实际上是我们构建的，而不是默认的。这有助于防止构建脚本更改无意中导致在引导构建中使用默认编译器。

然后作业清除所有构建产物，并开始正常构建 Roslyn.slnx，但指定 `/p:BootstrapBuildPath=...`。这会导致加载两个文件：

- [Bootstrap.props](https://github.com/dotnet/roslyn/blob/main/eng/targets/Bootstrap.props)：在默认编译器上加载引导编译器
- [Bootstrap.targets](https://github.com/dotnet/roslyn/blob/main/eng/targets/Bootstrap.targets)：验证引导编译器实际上被使用了

此测试还确保二进制日志和构建服务器日志被捕获在已发布的产物集中，以便于调查。

## 调查

调查正确性构建失败的第一步是下载日志文件。这些文件在已发布的产物中可用。

![已发布的产物](images/bootstrap-logs.png)

两个最有趣的文件是：

1. Build.Server.log：这是编译过程的文本日志。所有堆栈跟踪和服务器错误消息都会出现在此文件中
1. Build.binlog：这是使用引导编译器构建 dotnet/roslyn 产生的二进制日志。

构建服务器日志文件将包含特定请求到编译器失败的原因。在大多数情况下，搜索以下两个词之一可以直接找到失败。

第一个要搜索的词是 `"Debug.Assert"`（不含引号）。引导失败最常见的原因是编译期间 `Debug.Assert` 调用失败。这会导致异常被添加到带有完整堆栈跟踪的日志中。例如：

```txt
ID=VBCSCompiler TID=33: Debug.Assert failed with message: Fail: 
Stack Trace
   at Microsoft.CodeAnalysis.CommandLine.ExitingTraceListener.Exit(String originalMessage)
   at Microsoft.CodeAnalysis.CommandLine.ExitingTraceListener.WriteLine(String message)
   at System.Diagnostics.TraceInternal.Fail(String message)
   at System.Diagnostics.Debug.Assert(Boolean condition)
   at Microsoft.CodeAnalysis.GeneratorDriver..ctor(GeneratorDriverState state)
   ...
```

这些失败几乎总是由于正在测试的更改。本质上是更改以导致 `Assert` 失败的方式更新了编译器逻辑。偶尔也会因为 IDE 更改引入了触发编译器或分析器中潜在 bug 的编码模式，但这肯定是罕见的情况。

下一个要搜索的词是 `"Error "` （不含引号但保留空格）。每次服务器遇到错误并需要关闭时，这将被添加到日志中。

```txt
ID=MSBuild 60300 TID=3: Error Error: 'EndOfStreamException' 'Reached end of stream before end of read.' occurred during 'Reading response for d2c3aeac-bd8a-4251-bde0-2e11bbc57d13'
```

这类错误，当不与 `Debug.Assert` 失败配对时，几乎总是编译器服务器中的 bug。请联系编译器团队帮助追踪此类失败。

注意：当你遇到日志文件没有对构建失败的可操作描述时，强烈考虑发送一个 PR 来修复这个问题。这种方法是日志成为追踪引导失败的宝贵工具的原因。

### 本地重现

使用 `src/Tools/Replay` 在本地重现并附加调试器到引导构建失败相当简单，它可以使用已检出的 Roslyn 编译器源码重放 binlog。为此，使用 `-bl` 标志本地运行构建以获取二进制日志，然后使用 replay 重放二进制日志：

**Windows**

```powershell
.\build.ps1 -bl
dotnet run --framework net10.0 --project src\Tools\Replay\ artifacts\log\Debug\Build.binlog -w
```

**Linux**

```sh
./build.sh -bl
dotnet run --framework net10.0 --project src/Tools/Replay/ artifacts/log/Debug/Build.binlog -w
```

`-w` 标志将导致 replay 打印编译器的进程 ID 并等待直到你按下一个键，这样你就可以将调试器附加到进程并为你感兴趣的异常设置断点。

## 调试

要在本地调试引导构建失败，请按以下步骤操作。

第一步是禁用 `ExitingTraceListener`。这对于 CI 很重要，在 CI 中编译器需要在 `Debug.Assert` 失败时崩溃，而不是弹出会挂起 CI 的对话框。然而，在本地调试时，开发者需要 `Debug.Assert` 弹出对话框的行为。要禁用 `ExitingTraceListener`，注释掉以下行：

https://github.com/dotnet/roslyn/blob/d73d31cbccb9aa850f3582afb464b709fef88fd7/src/Compilers/Server/VBCSCompiler/VBCSCompiler.cs#L22

然后只需在本地运行引导构建，等待 `Debug.Assert` 触发并弹出对话框。从那里你可以附加到 VBCSCompiler 进程并调试问题。

```cmd
> Build.cmd -bootstrap
```
