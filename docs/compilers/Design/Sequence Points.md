序列点设计
======================

序列点被写入 PDB/符号文件，用于告知调试器 IL 与源代码之间的对应关系。序列点由 IL 偏移量和关联的文档 span 组成。  
当调试器停在某条 IL 指令上时，它从该 IL 偏移量向前找到第一个序列点，然后找到关联的 span 并为用户高亮显示。

序列点通常在 Roslyn 中用于语句级别。它们被限制只能出现在代码中求值栈为空的位置。

我们还有[基础设施](https://github.com/dotnet/roslyn/blob/main/docs/compilers/CSharp/Expression%20Breakpoints.md)，通过将表达式溢出到语句中来支持表达式断点。  
这用于 `switch` 表达式，允许调试器先在操作数处暂停，然后再在所选分支处暂停。

调试器在跳转指令之后会自动暂停。因此合成的标签通常需要一个序列点。否则，任何之前已有的序列点都会被使用，这会导致前一条语句被高亮显示。  
隐藏序列点可用于此目的。它们是没有关联 span 的序列点。当调试器遇到隐藏序列点时，会恢复执行而不是暂停。

在整个合成方法都应被调试器跳过的情况下，可以用 `[DebuggerHidden]` 标记该方法。这用于 `MoveNext()` 方法之外的状态机方法。

`VerifyIL` 测试助手方便地支持将 IL 与序列点（以源代码片段形式展示）混合显示：
```il
    // sequence point: {
    IL_0045:  nop
    // sequence point: Write("1 ");
    IL_0046:  ldstr      "1 "
    IL_004b:  call       "void System.Console.Write(string)"
    IL_0050:  nop
    // sequence point: await System.Threading.Tasks.Task.CompletedTask;
    IL_0051:  call       "System.Threading.Tasks.Task System.Threading.Tasks.Task.CompletedTask.get"
    IL_0056:  callvirt   "System.Runtime.CompilerServices.TaskAwaiter System.Threading.Tasks.Task.GetAwaiter()"
    IL_005b:  stloc.1
    // sequence point: <hidden>
    IL_005c:  ldloca.s   V_1
```
