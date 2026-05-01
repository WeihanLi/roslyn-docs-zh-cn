# 引用程序集

引用程序集是仅包含元数据的程序集，其中包含保持消费者编译时行为所需的最少元数据量（尽管诊断信息可能会受到影响）。

编译器可能会在以后的版本中删除更多元数据，前提是确认此操作是安全的（即遵守上述原则）。

## 场景
共有 4 个场景：

1. 传统场景：程序集作为主输出被生成（`/out` 命令行参数，或 `Compilation.Emit` API 中的 `peStream` 参数）。
2. IDE 场景：通过 `Emit` API 生成仅元数据程序集，仍作为主输出。在 C# 7.1 之后，IDE 对于在编译中存在错误时也希望获取仅元数据程序集。
3. CoreFX 场景：仅生成 ref 程序集作为主输出（`/refonly` 命令行参数）。
4. MSBuild 场景：这是新场景，同时生成真实程序集作为主输出以及 ref 程序集作为次输出（`/refout` 命令行参数，或 `Emit` 中的 `metadataPeStream` 参数）。


## ref 程序集的定义
仅元数据程序集会将其方法体替换为单个 `throw null` 体，但包含除匿名类型外的所有成员。使用 `throw null` 体（而非无体）的原因是使 PEVerify 能够运行并通过（从而验证元数据的完整性）。

ref 程序集包含程序集级别的 `ReferenceAssembly` 特性。该特性可在源代码中指定（这样就不需要合成它）。由于该特性，运行时将拒绝加载 ref 程序集执行（但仍可以以仅反射模式加载）。某些工具可能受到影响，需要更新（例如 `sgen.exe`）。

ref 程序集进一步从仅元数据程序集中删除元数据（私有成员）：

- ref 程序集仅包含其 API 表面所需的引用。真实程序集可能有与特定实现相关的额外引用。例如，`class C { private void M() { dynamic d = 1; ... } }` 的 ref 程序集不会引用 `dynamic` 所需的任何类型。
- 私有函数成员（方法、属性和事件）被删除。如果没有 `InternalsVisibleTo` 特性，则对内部函数成员也执行相同操作。
- 但所有类型（包括私有或嵌套类型）都保留在 ref 程序集中。所有特性都保留（包括内部的），以及其（内部）构造函数。
- 所有虚方法都保留。显式接口实现保留。显式实现的属性和事件保留，因为其访问器是虚拟的（因此保留）。
- 保留 struct 的所有字段。（这是 C# 7.1 后续改进的候选项）
- 命令行中包含的任何资源不会生成到 ref 程序集中（无论是通过 `/refout` 还是 `/refonly` 生成）。（此问题在 dev16 中已修复）

## API 变更

### 命令行
向 `csc.exe` 和 `vbc.exe` 添加了两个互斥的命令行参数：
- `/refout`
- `/refonly`

`/refout` 参数指定 ref 程序集应输出到的文件路径。这对应 `Emit` API 中的 `metadataPeStream`（详见下文）。ref 程序集的文件名通常应与主程序集的文件名相匹配。MSBuild 使用的推荐约定是将 ref 程序集放在相对于主程序集的 "ref/" 子文件夹中。

`/refonly` 参数是一个标志，指示应输出 ref 程序集而不是实现程序集作为主输出。

`/refonly` 参数不允许与 `/refout` 参数一起使用，因为将主输出和次输出都设为 ref 程序集没有意义。此外，`/refonly` 参数会静默禁用 PDB 输出，因为 ref 程序集无法执行。

`/refonly` 参数对应于 `Emit` API 中 `EmitMetadataOnly` 设为 `true` 且 `IncludePrivateMembers` 设为 `false`（详见下文）。

`/refonly` 和 `/refout` 均不允许与网络模块（`/target:module`、`/addmodule` 选项）一起使用。

从命令行编译要么同时生成两个程序集（实现和 ref），要么都不生成。不存在"部分成功"场景。

当编译器生成文档时，`/refonly` 或 `/refout` 参数不会对其产生影响。这在将来可能会改变。

`/refout` 选项的主要目的是加速增量构建场景，因此当前实现中此标志生成的 ref 程序集可能包含比 `/refonly` 更多的元数据（例如匿名类型）。这是 C# 7.1 后续改进的候选项。

### CscTask/CoreCompile
`CoreCompile` 目标支持一个名为 `IntermediateRefAssembly` 的新输出，与现有的 `IntermediateAssembly` 对应。

`Csc` 任务支持一个名为 `OutputRefAssembly` 的新输出，与现有的 `OutputAssembly` 对应。
这两者基本上都映射到 `/refout` 命令行参数。

还提供了一个名为 `CopyRefAssembly` 的附加任务，与现有的 `Csc` 任务一起提供。它接受 `SourcePath` 和 `DestinationPath`，通常将文件从源路径复制到目标路径。但如果它能确定两个文件的内容匹配（通过比较其 MVID，详见下文），则目标文件保持不变。

附带说明，`CopyRefAssembly` 使用与 `Csc` 和 `Vbc` 相同的程序集解析/重定向技巧，以避免 `System.IO.FileSystem` 的类型加载问题。

### CodeAnalysis API
在 C# 7.1 之前，已经可以通过使用 `EmitOptions.EmitMetadataOnly` 来生成仅元数据程序集，该选项在具有跨语言依赖项的 IDE 场景中使用。

在 C# 7.1 中，编译器现在也遵循 `EmitOptions.IncludePrivateMembers` 标志。与 `EmitMetadataOnly` 或 `Emit` 中的 `metadataPeStream` 结合使用时，将生成 ref 程序集。

使用 `EmitMetadataOnly` 时不会编译方法体。甚至用于检测缺少方法体（`void M();`）的诊断也会从声明诊断中过滤掉，因此此类代码将成功使用 `EmitMetadataOnly` 生成。

之后，`EmitOptions.TolerateErrors` 标志将允许也生成错误类型。
`Emit` 已被修改，在生成 ref 程序集时会生成一个名为 ".mvid" 的新 PE 节，其中包含 MVID 的副本。这使 `CopyRefAssembly` 能够轻松地从 ref 程序集中提取并比较 MVID。

回到 4 个驱动场景：
1. 对于常规编译，`EmitMetadataOnly` 保持 `false`，不向 `Emit` 传入 `metadataPeStream`。
2. 对于 IDE 场景，`EmitMetadataOnly` 设为 `true`，但 `IncludePrivateMembers` 保持 `true`。
3. 对于 CoreFX 场景，使用 ref 程序集源代码作为输入，`EmitMetadataOnly` 设为 `true`，`IncludePrivateMembers` 设为 `false`。
4. 对于 MSBuild 场景，`EmitMetadataOnly` 保持 `false`，传入 `metadataPeStream`，`IncludePrivateMembers` 设为 `false`。

### 确定性

我们建议始终将确定性与引用程序集一起使用。这可以最大程度地减少 ref 程序集的变化频率，从而最大化其带来的好处。

也就是说，即使未设置确定性，ref 程序集的编译在默认情况下也是[基本确定性的](http://blog.paranoidcoding.com/2016/04/05/deterministic-builds-in-roslyn.html)。主要的例外情况是使用带通配符的 `AssemblyVersionAttribute`（例如 `[assembly: System.Reflection.AssemblyVersion("1.0.*")]`）。在这种情况下，编译必然是非确定性的，因此 ref 程序集不会带来任何好处。

## MSBuild

* `ProduceReferenceAssembly`（布尔值）控制是否创建传递给编译器任务的项目（从而传递 `/refout:`）。需要选择启用。建议同时设置 `Deterministic` 以获得最佳效果（详见上文）。不能与 `ProduceOnlyReferenceAssembly` 一起使用。
* `ProduceOnlyReferenceAssembly`（布尔值）控制是否向编译器传递 `/refonly`。需要选择启用。不能与 `ProduceReferenceAssembly` 一起使用。
* 如果遇到使用引用程序集的问题，可以将布尔属性 `CompileUsingReferenceAssemblies` 设为 `false`，以避免使用 ref 程序集，即使你引用的项目会生成它们。此属性默认未设置，且仅在设为 `false` 时检查。它仅用于提供紧急出口；遇到 bug 的用户可以将其设为 `false` 以绕过新代码路径。

## 未来工作
如上所述，C# 7.1 之后可能有进一步的改进：
- 进一步减少 `/refout` 生成的 ref 程序集中的元数据，使其与 `/refonly` 生成的一致。
- 控制内部成员，使其即使在 `InternalsVisibleTo` 的情况下也不包含在内（生成公共 ref 程序集）。
- 在方法体之外存在错误时也生成 ref 程序集（当设置 `EmitOptions.TolerateErrors` 时生成错误类型）。
- 当编译器生成文档时，可以对内容进行过滤以匹配进入主输出的 API。换句话说，使用 `/refonly` 参数时，文档可以被过滤。

## 未解决问题

## 相关问题
- 从命令行和 msbuild 生成 ref 程序集（https://github.com/dotnet/roslyn/issues/2184）
- 细化引用程序集中的内容以及哪些诊断会阻止生成（https://github.com/dotnet/roslyn/issues/17612）
- [私有成员是否属于 API 表面？](https://blog.paranoidcoding.org/2016/02/15/are-private-members-api-surface.html)
- MSBuild 工作项和设计说明（https://github.com/Microsoft/msbuild/issues/1986）
- project-system 中的快速最新检查（https://github.com/dotnet/project-system/issues/2254）
