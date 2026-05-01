# 源生成器

## 摘要

> **警告**：实现 `ISourceGenerator` 的源生成器已被弃用，
> 改为使用[增量生成器](incremental-generators.md)。

源生成器旨在实现_编译时元编程_，即可以在编译时创建并添加到编译中的代码。源生成器将能够在运行之前读取编译的内容，以及访问任何_附加文件_，使生成器能够内省用户 C# 代码和生成器特定的文件。

> **注意**：此提案与[先前的生成器设计](generators.md)是分开的

### 高级设计目标

- 生成器生成一个或多个字符串，这些字符串表示要添加到编译中的 C# 源代码。
- 明确地_仅添加_。生成器可以向编译添加新的源代码，但**不能**修改现有的用户代码。
- 可以产生诊断。当无法生成源时，生成器可以通知用户问题。
- 可以访问_附加文件_，即非 C# 源文本。
- 运行_无序_，每个生成器将看到相同的输入编译，无法访问其他源生成器创建的文件。
- 用户通过程序集列表指定要运行的生成器，就像分析器一样。

## 实现

在最简单的级别上，源生成器是 `Microsoft.CodeAnalysis.ISourceGenerator` 的实现

```csharp
namespace Microsoft.CodeAnalysis
{
    public interface ISourceGenerator
    {
        void Initialize(GeneratorInitializationContext context);
        void Execute(GeneratorExecutionContext context);
    }
}
```

生成器实现在外部程序集中定义，使用与诊断分析器相同的 `-analyzer:` 选项传递给编译器。实现需要使用 `Microsoft.CodeAnalysis.GeneratorAttribute` 属性进行注释。

程序集可以包含诊断分析器和源生成器的混合。由于生成器是从外部程序集加载的，因此生成器不能用于构建其定义所在的程序集。

`ISourceGenerator` 有一个 `Initialize` 方法，由主机（IDE 或命令行编译器）恰好调用一次。`Initialize` 传递一个 `GeneratorInitializationContext` 实例，生成器可以使用它来注册一组回调，这些回调会影响将来的生成过程如何发生。

主要的生成过程通过 `Execute` 方法进行。`Execute` 传递一个 `GeneratorExecutionContext` 实例，提供对当前 `Compilation` 的访问，并允许生成器通过添加源和报告诊断来改变结果输出 `Compilation`。

生成器还能够通过 `AdditionalFiles` 集合访问传递给编译器的任何 `AnalyzerAdditionalFiles`，允许基于不仅仅是用户的 C# 代码做出生成决策。

```csharp
namespace Microsoft.CodeAnalysis
{
    public readonly struct GeneratorExecutionContext
    {
        public ImmutableArray<AdditionalText> AdditionalFiles { get; }

        public CancellationToken CancellationToken { get; }

        public Compilation Compilation { get; }

        public ISyntaxReceiver? SyntaxReceiver { get; }

        public void ReportDiagnostic(Diagnostic diagnostic) { throw new NotImplementedException(); }

        public void AddSource(string fileNameHint, SourceText sourceText) { throw new NotImplementedException(); }
    }
}
```

假设某些生成器会想要生成多个 `SourceText`，例如对附加文件进行 1:1 映射。`AddSource` 的 `fileNameHint` 参数旨在解决这个问题：

1. 如果生成的文件被发出到磁盘，能够放置一些区分文本可能很有用。例如，如果您有两个 `.resx` 文件，生成名为 `ResxGeneratedFile1.cs` 和 `ResxGeneratedFile2.cs` 的文件不会非常有用——您希望它是类似 `ResxGeneratedFile-Strings.cs` 和 `ResxGeneratedFile-Icons.cs` 这样的，如果您有两个分别命名为"Strings"和"Icons"的 `.resx` 文件。

2. IDE 需要某种"稳定"标识符的概念。源生成器为 IDE 创造了一些有趣的问题：例如，用户会想要能够在生成的文件中设置断点。如果源生成器输出多个文件，我们需要知道哪个是哪个，以便我们知道断点属于哪个文件。当然，如果输入改变，源生成器可以停止发出文件（如果您删除了 `.resx`，则与之关联的生成文件也会消失），但这给了我们一些控制权。

这被称为"提示"，因为编译器被隐式允许以任何最终需要的方式控制文件名，如果两个源生成器给出相同的"提示"，它仍然可以根据需要使用任何前缀/后缀来区分它们。

### IDE 集成

支持生成器的更复杂方面之一是在 Visual Studio 中启用高保真体验。为了确定代码正确性，预期所有生成器都必须已运行。显然，在每次击键时运行每个生成器并仍然保持 IDE 内可接受的性能水平是不切实际的。

#### 渐进式复杂性选择加入

相反，预期源生成器将在 IDE 启用方面采用"选择加入"方法。

默认情况下，仅实现 `ISourceGenerator` 的生成器不会看到 IDE 集成，只在构建时正确。根据与第一方客户的对话，有几种情况这就足够了。

但是，对于诸如代码优先 gRPC 以及特别是 Razor 和 Blazor 等场景，IDE 需要能够在编辑这些文件类型时即时生成代码，并将更改近乎实时地反映回 IDE 中的其他文件。

建议是有一组可以选择性实现的高级回调，这将允许 IDE 查询生成器以决定在任何特定编辑的情况下需要运行什么。

例如，在保存第三方文件后导致生成运行的扩展可能如下所示：

```csharp
namespace Microsoft.CodeAnalysis
{
    public struct GeneratorInitializationContext
    {
        public void RegisterForAdditionalFileChanges(EditCallback<AdditionalFileEdit> callback){ }
    }
}
```

这将允许生成器在初始化期间注册回调，每次附加文件更改时都会调用该回调。

预期会有各种级别的选择加入，可以添加到生成器以实现其所需的特定性能级别。

这些确切的 API 是什么样子仍然是一个开放的问题，预计我们需要原型化一些真实世界的生成器，然后才能知道它们的精确形状。

### 输出文件

希望生成的源文本在生成后可供检查，无论是作为创建生成器的一部分还是查看第三方生成器生成了什么代码。

默认情况下，生成的文本将保存到 `CommandLineArguments.OutputDirectory` 中的 `GeneratedFiles/{GeneratorAssemblyName}` 子文件夹。来自 `GeneratorExecutionContext.AddSource` 的 `fileNameHint` 将用于创建唯一名称，如果需要则应用适当的冲突重命名。例如，在 Windows 上，来自 `MyGenerator.dll` 的 C# 项目调用 `AddSource("MyCode", ...);` 可能会保存为 `obj/debug/GeneratedFiles/MyGenerator.dll/MyCode.cs`。

文件输出对于命令行或基于 IDE 的生成的正确功能不是必需的，如果需要可以完全禁用。IDE 将在生成的源文本的内存副本上工作（用于"查找所有引用"、断点等），并定期将任何更改刷新到磁盘。

为了支持用户希望生成源文本然后将生成的文件提交到源代码管理的用例，我们将允许通过适当的命令行开关和匹配的 MSBuild 属性更改生成文件的位置（命名尚未确定）。

在这些情况下，用户可以决定是否希望将来再次生成文件（在这种情况下它们仍然会生成，但输出到源代码控制的位置），或者删除生成器并将操作作为一次性步骤执行。

例如，在基于磁盘的生成文件中设置断点的操作将如何工作目前是一个开放的问题。

待定：我们如何保存 PDB/Source link 等？

### 第三方语言的编辑体验

源生成器将启用的一个有趣场景本质上是 C# 在其他语言中的"嵌入"（反之亦然）。这就是 Razor 今天的工作方式，Razor 团队在 Visual Studio 中维护着重要的语言服务投资来启用它。

此项目的一个可能目标是找到一种通用的方式来表示这一点：这将允许 Razor 团队减少他们的工具投资，同时允许第三方以相对便宜的方式启用相同类型的体验（包括"转到定义"、"查找所有引用"等）。

当前的想法是为生成器提供某种形式的"旁通道"。当生成器发出源文本时，它会指示这是从原始文档的哪个位置生成的。这将允许编译器 API 跟踪例如生成的 `Symbol` 具有表示第三方源文本范围的 `OriginalDefinition`（例如 `.cshtml` 文件中的 Razor 标签）。

我们讨论过通过 `#pragma` 将其直接嵌入到源文本中，但这需要语言更改并将功能限制为特定版本的 C#。其他考虑可以是特殊格式的注释或 `#if FALSE --` 块。一般来说，"旁通道"方法似乎比生成文本中的特殊语法更可取。

这不一定是源生成器成功所需的目标；如果证明不可行，可以更新 Razor 的语言服务以使用源生成器，但这当然是我们作为工作的一部分要考虑的事情。

### MSBuild 集成

预期生成器将需要某种形式的配置系统，我们打算允许某些属性从 MSBuild 流过来以促进这一点。

> **注意**：这仍在设计中，可能会有变化。


### 性能目标

最终，该功能的性能在某种程度上取决于客户编写的生成器的性能。渐进式选择加入和默认仅构建时将允许 IDE 缓解第三方生成器造成的许多潜在性能问题。但是，仍然存在第三方生成器会导致 IDE 性能不可接受问题的风险，功能的设计需要牢记这一点。

对于第一方生成器，特别是 Razor 和 Blazor，我们的目标至少是匹配用户今天看到的现有性能。预期即使是天真的基于生成器的实现也会比现有工具执行得更快，由于更少的通信开销和重复工作，但改善这些体验的速度不是此项目的主要目标。

### 语言更改

此设计目前不建议改变语言，它纯粹是编译器功能。先前的源生成器设计引入了 `replace` 和 `original` 关键字。此提案删除了这些，因为生成的源是纯粹附加的，因此不需要它们。我们预期大多数场景都可以使用现有的 `partial` 定义；作为 V1，我们预计将以这种状态发布。如果后来显示具体场景无法使用 V1 方法实现，我们将考虑允许修改作为 V2。

## 用例

我们已经确定了几个将受益于源生成器的第一方和第三方候选者：

- ASP.Net：改善启动时间
- Blazor 和 Razor：大幅减少工具负担
- Azure Functions：启动期间的正则表达式编译
- Azure SDK
- [gRPC](https://docs.microsoft.com/en-us/aspnet/core/grpc/?view=aspnetcore-3.1)
- Resx 文件生成
- [System.CommandLine](https://github.com/dotnet/command-line-api)
- 序列化器
- [SWIG](http://www.swig.org/)


## 讨论/开放问题/待办事项：

**ISourceGenerator 的接口与类**：

我们讨论了这应该是接口还是类。分析器选择使用抽象基类，但我们不确定最终会需要什么，因为最终我们只有一个方法。保持它作为接口也更自然，因为我们有其他实现此接口的接口，用于可选的轻量级支持。

**IDependsOnCompilationGenerator**：

我们确实讨论过是否应该有一个 IDependsOnCompilationGenerator 来正式声明您实际上使用了编译。毕竟，如果您不使用编译，那么我们知道您在 IDE 中的性能大大简化了。但是，我们用于读取附加文件的每个场景也都需要编译，所以我们只是不确定这会带来什么。

**生成文件中的断点**：

我们是否将其映射回内存文件？

**生成器应该是推还是拉**：

源生成器是基于拉的，分析器是基于推的（基于注册）。我们也应该为生成器使用基于推的模型吗？

- 如果我们采用基于推的模型，遍历树应该确保继续为尽可能多的节点产生事件，即使有错误，因为生成器通常会在存在错误的情况下工作

- 我们今天用于分析器的事件可能需要更多工作才能产生，因为我们期望分析器在完整编译期间运行，而生成器可能甚至不想构建符号表

- 渐进式性能选择加入模型在基于推的模型中可能效果更好，因为您只会注册您关心的事情

**我们应该与分析器类型层次结构共享更多吗？**：

我们仍然需要区分分析器和生成器，因为它们将在不同时间生成（生成器诊断仅在第一次编译时，分析器诊断仅在第二次编译时）

**我们能预测一些示例客户（Razor？）需要多久运行一次生成器吗？**：

他们现在无法预测这一点，将计时器纳入其当前生成使得预测仅基于事件的生成的后果非常困难

**我们有最重要客户的优先级列表吗？**：

没有，我们应该按优先级排出以优先考虑功能。

**安全审查**：

生成器是否创建了分析器和 nuget 尚未造成的任何新安全风险？
