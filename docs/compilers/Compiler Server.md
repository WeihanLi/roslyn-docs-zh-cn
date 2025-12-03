编译器服务器
===============

作为性能优化，Roslyn C# 和 VB 编译器支持使用服务器进程执行编译，而不是在进程内执行编译。工程重点关注服务器的三个性能优势：

1. 重用单个进程进行编译可以分摊编译器代码的 JIT 编译成本。这通常是最大的单一性能优势。
2. Roslyn 要求将程序集引用水合成丰富的对象模型才能执行编译。编译器服务器持有一个固定大小的元数据符号缓存，如果大多数编译共享相当大部分的元数据引用（例如 mscorlib），这可以帮助缩短编译时间。
3. 在 Windows 上，进程创建本身是昂贵的，因此直接从 MSBuild 分派编译可以节省 csc.exe 的创建。

## 用法

使用编译器服务器的主要方式是通过 `Microsoft.Build.Tasks.CodeAnalysis` MSBuild 任务，该任务随 `dotnet` SDK 和 Visual Studio 一起提供。构建任务包含一个构建客户端，它不是执行 `csc.exe` 或 `vbc.exe`，而是创建 `VBCSCompiler.exe` 进程并将命令行参数直接分派给服务器。

如果您不使用 MSBuild，还有一个标志 `/shared`，可以直接传递给 `csc.exe`，这会导致 `csc.exe` 充当客户端并将实际编译分派给服务器进程。在此模式下，`csc.exe` 本身不执行任何计算，它只是将命令行分派给服务器进程（如果不存在则启动一个），然后报告编译结果（包括产生的任何警告/错误）。

重要的是，服务器是一种性能优化。如果服务器编译由于任何原因失败，编译器将回退到独立编译。对于 MSBuild，这意味着创建标准的 `csc.exe` 进程。对于 `/shared`，这意味着继续在进程内编译，就像没有传递 `/shared` 一样。在这种情况下不会产生任何诊断，因为任何给定的服务器编译失败都有非常常见的环境原因。

这也意味着杀死编译器服务器进程始终是安全的，因为客户端将始终优雅地恢复。

## 架构

Roslyn 支持客户端-服务器协议，其中多个客户端可以分派给多个服务器，多个服务器可以在机器上并发运行。客户端和服务器必须在同一台机器上运行，因为客户端和服务器只交换命令行，而不是实际资产。

## 故障排除

要诊断服务器编译的问题，您可以启用一个环境变量来写入诊断日志文件：

```
(UNIX): export RoslynCommandLineLogFile=/path/to/log/file
(Windows CMD，使用目录路径): setx RoslynCommandLineLogFile="C:\path\to\log"
(Windows CMD，使用文件路径): setx RoslynCommandLineLogFile="C:\path\to\log\file.log"
```

此文件包含客户端和服务器的诊断日志，您可以使用每个诊断行的 `PID` 标签来区分。示例输出应该类似于：

```
--- PID=90433 TID=16 Ticks=1048119217: Attempt to open named pipe 'angocke.F.MW9E3dnrZddlsgI8MNFfpJyup'
--- PID=90433 TID=16 Ticks=1048119217: Attempt to connect named pipe 'angocke.F.MW9E3dnrZddlsgI8MNFfpJyup'
--- PID=90436 TID=1 Ticks=244465170: Keep alive timeout is: 600000 milliseconds.
--- PID=90436 TID=1 Ticks=244465182: Constructing pipe 'angocke.F.MW9E3dnrZddlsgI8MNFfpJyup'.
--- PID=90436 TID=1 Ticks=244465182: Successfully constructed pipe 'angocke.F.MW9E3dnrZddlsgI8MNFfpJyup'.
```

这里 PID `90433` 是客户端，因为它正在尝试打开连接（命名管道），而 PID `90436` 是服务器，因为它正在构造命名客户端。如果成功，结果看起来像这样：

```
--- PID=90436 TID=8 Ticks=244476082: ****C# Compilation complete.
****Return code: 0
****Output:
Microsoft (R) Visual C# Compiler version 2.9.0.63018 (e3fa7d6a)
Copyright (C) Microsoft Corporation. All rights reserved.
```

否则，您可能能够从其他输出（如异常消息）中发现问题所在。
