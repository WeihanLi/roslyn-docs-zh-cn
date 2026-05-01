API 破坏性变更
====

# 版本 1.1.0

### 移除了 VisualBasicCommandLineParser.ctor
在工具集更新期间，我们注意到 `VisualBasicCommandLineParser` 上的构造函数是 `public` 的。这反过来使得 `CommandLineParser` 的许多 `protected` 成员成为 API 表面的一部分，因为它为外部客户提供了一个继承路径。

这些成员从来就不是预期支持的 API 表面的一部分。解析器的创建应该通过 `Default` 单例属性来完成。我们认为在这里破坏任何客户的风险很小，因此我们决定移除这个 API。

PR: https://github.com/dotnet/roslyn/pull/4169

### 更改了 Simplifier 方法以抛出 ArgumentNullExceptions
更改了 Simplifier.ReduceAsync、Simplifier.ExpandAsync 和 Simplifier.Expand 方法，当传入任何非可选的可空参数时抛出 ArgumentNullExceptions。以前，用户在同步方法中会得到 NullReferenceException，在异步方法中会得到包含 NullReferenceException 的 AggregateException。

PR: https://github.com/dotnet/roslyn/pull/5144

# 版本 1.3.0

### 将同时标记为 public 和 private 标志的方法视为 private

该场景是加载一个程序集，其中某些方法、字段或嵌套类型的可访问性标志设置为 7（所有三个位都设置），这意味着 public 和 private。
修复后，这些标志被加载为表示 private。
兼容性变更是我们用编译时成功和运行时失败（本机编译器）换取编译时错误（恢复 v1.2 的行为）。

详细信息如下：

- 本机编译器成功编译方法和字段的情况（那些只产生运行时错误 System.TypeLoadException: Invalid Field Access Flags），并在嵌套类型上报告可访问性错误。
- 1.2 编译器生成错误：
```
error BC30390: 'C.Private Overloads Sub M()' is not accessible in this context because it is 'Private'.
error BC30389: 'C.F' is not accessible in this context because it is 'Private'.
error BC30389: 'C.C2' is not accessible in this context because it is 'Protected Friend'.
error BC30390: 'C2.Private Overloads Sub M2()' is not accessible in this context because it is 'Private'.
```
- 1.3 编译器崩溃。
- 修复后，再次生成与 1.2 相同的错误。

PR: https://github.com/dotnet/roslyn/pull/11547

### 不发出错误的 DateTimeConstant，并将错误的 DateTimeConstant 加载为默认值

该更改以两种方式影响兼容性：

- 当加载无效的 DateTimeConstant(-1) 时，编译器将使用 default(DateTime) 代替，而本机编译器会生成执行失败的代码。
- DateTimeConstant(-1) 在检查是否指定了两个默认值时仍然会被计算。编译器将产生错误，而不是成功（并生成带有两个属性的 IL）。

PR: https://github.com/dotnet/roslyn/pull/11536

# 版本 4.1.0

### 不能再从 CompletionService 和 CompletionServiceWithProviders 继承

Microsoft.CodeAnalysis.Completion 和 Microsoft.CodeAnalysis.Completion.CompletionServiceWithProviders 的构造函数现在是 internal 的。
Roslyn 不支持为任意语言实现完成功能。

# 版本 4.2.0

### 不能再从 QuickInfoService 继承

Microsoft.CodeAnalysis.QuickInfoService 的构造函数现在是 internal 的。
Roslyn 不支持为任意语言实现完成功能。

### `Microsoft.CodeAnalysis.CodeStyle.NotificationOption` 现在是不可变的

所有属性设置器现在都会抛出异常。

# 版本 4.4.0

从磁盘读取源文件内容时发生错误时，不再调用 `Workspace.OnWorkspaceFailed`。

`TextLoader.LoadTextAndVersionAsync(Workspace, DocumentId, CancellationToken)` 的 `Workspace` 和 `DocumentId` 参数已弃用。
该方法现在接收 `null` 的 `Workspace` 和 `DocumentId`。

# 版本 4.5.0

如果在独立的 `IParameterSymbol` 上使用 `SymbolDisplayFormat.CSharpErrorMessageFormat` 和 `CSharpShortErrorMessageFormat`，现在默认包含参数名称。
例如，`void M(ref int p)` 中的参数 `p` 以前格式化为 `"ref int"`，现在格式化为 `"ref int p"`。

# 版本 4.7.0

### 在 `IParameterSymbol` 上调用时 `SymbolDisplayFormat` 包含参数名称

所有 `SymbolDisplayFormat`（预定义的和用户创建的）现在如果在独立的 `IParameterSymbol` 上使用，默认都包含参数名称，以与预定义格式保持一致（参见上面版本 4.5.0 的破坏性变更）。

### 当修改的输入产生新输出时更改了 `IncrementalStepRunReason`

当步骤的输入以产生新输出的方式被修改时，`IncrementalGeneratorRunStep.Outputs` 以前在 `Reason` 中包含 `IncrementalStepRunReason.Modified`。
现在原因将更准确地报告为 `IncrementalStepRunReason.New`。

# 版本 4.8.0

### 在非 Windows 中更改了 `Assembly.Location` 行为

`Assembly.Location` 的值以前保存分析器或源生成器从磁盘加载的位置。这可能是原始位置或影子复制位置。在 4.8 中，在非 Windows 平台上运行时，某些情况下这将是 `""`。这是由于编译器服务器使用 `AssemblyLoadContext.LoadFromStream` 而不是从磁盘加载程序集。

这在其他加载场景中已经可能发生，但此更改将其移入主线构建场景。

### SyntaxNode 序列化的弃用警告

将 SyntaxNode 序列化/反序列化到/从 Stream 的功能已被弃用。此代码仍然存在于 Roslyn 中，但尝试调用执行这些功能的 API 将导致报告"已过时"警告。Roslyn 的未来版本将完全移除此功能。此功能只能在将节点写入流的主机中使用，并在同一进程实例中稍后读回。它不能用于跨进程通信，或用于将节点持久化到磁盘以便稍后由新的主机会话读取。此功能最初存在于 Roslyn 托管在具有有限地址空间的 32 位进程中的时代。这不再是主线支持的场景。客户端可以通过持久化节点的文本并在需要时解析它来获得类似的功能。

PR: https://github.com/dotnet/roslyn/pull/70365

# 版本 4.9.0

### SyntaxNode 序列化的过时和移除

继续 4.8.0 中发生的弃用（参见上面的信息）。在 4.9.0 中，此功能现已完全移除，如果使用这些 API，将发出过时错误，并在运行时抛出异常。

PR: https://github.com/dotnet/roslyn/pull/70277

### `Microsoft.CodeAnalysis.Emit.EmitBaseline.CreateInitialBaseline` 方法的更改

添加了新的必需参数 `Compilation`。没有此参数的现有重载不再工作并抛出 `NotSupportedException`。

### `Microsoft.CodeAnalysis.Emit.SemanticEdit` 构造函数的更改

传递给构造函数的 `preserveLocalVariables` 值不再使用。
