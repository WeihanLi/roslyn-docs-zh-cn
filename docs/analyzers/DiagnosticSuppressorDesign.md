# 功能请求

https://github.com/dotnet/roslyn/issues/30172：_警告的程序化抑制_

为平台/库作者提供一种能力，使其可以编写简单的、具有上下文感知的编译器扩展，以程序化方式抑制已报告的特定分析器和/或编译器诊断实例——这些实例在平台/库的上下文中始终属于误报。

# 可程序化抑制的诊断

满足以下**所有**条件的分析器/编译器诊断，可被视为程序化抑制的候选：

1. _默认非错误_：诊断的 [DefaultSeverity](http://source.roslyn.io/#Microsoft.CodeAnalysis/Diagnostic/Diagnostic.cs,8e27a878b4d6e40c) **不是** [DiagnosticSeverity.Error](http://source.roslyn.io/#Microsoft.CodeAnalysis/Diagnostic/DiagnosticSeverity.cs,f771032fb5a00c1c)。
2. _必须可配置_：诊断**未**带有 [WellKnownDiagnosticTags.NotConfigurable](http://source.roslyn.io/#Microsoft.CodeAnalysis/Diagnostic/WellKnownDiagnosticTags.cs,207e57dd0b96bd4b) 自定义标签，表示其严重性可配置。
3. _无现有源代码抑制_：诊断**未**通过 pragma/suppress message 特性在源代码中被抑制。

# 核心设计

1. _面向平台/库作者：_ 公开新的 "DiagnosticSuppressor" 公开 API，用于编写此类编译器扩展。抑制器的 API 契约（[最后一节](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#detailed-api-proposal-with-documentation-comments-from-the-draft-pr)中包含带文档注释的详细 API）：
   1. 以声明方式提供所有可被其抑制的分析器和/或编译器诊断 ID。
   2. 对于每个可抑制的诊断 ID，提供唯一的抑制 ID 和理由说明。这些信息是实现对每项抑制进行正确诊断和配置所必需的（[后文](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#example-experience-with-screenshots)有详细介绍）。
   3. 为带有这些可抑制诊断 ID 的已报告分析器和/或编译器诊断注册回调。回调可分析诊断位置的语法和/或语义，并报告抑制。
   4. DiagnosticSuppressor **不能**注册任何其他分析回调，因此无法报告新的诊断。
2. _面向最终用户：_
   1. 针对此类平台/库开发时体验无缝衔接——用户不会在其开发上下文中看到来自分析器/编译器的误报。
   2. 最终用户无需为此类上下文特定的误报手动添加和/或维护通过 pragma/SuppressMessageAttribute 实现的源代码抑制。
   3. 最终用户仍对抑制器拥有最终控制权（[后续章节](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#example-experience-with-screenshots)中有具体说明）：
      1. 审计：诊断抑制会在命令行中以"Info"诊断的形式记录，最终用户可在详细构建日志或 msbuild binlog 中审计每项抑制。此外，它们也会以已抑制诊断的形式记录在 [/errorlog SARIF 文件](https://github.com/dotnet/roslyn/blob/master/docs/compilers/Error%20Log%20Format.md)中。
      2. 配置：每项诊断抑制均有关联的抑制 ID。该 ID 在抑制的"Info"诊断中有明确标注，最终用户可通过简单的命令行参数或规则集条目禁用特定抑制 ID 下的一批抑制。
3. _面向分析器驱动：_
   1. 分析器/编译器与抑制器的执行顺序：对于命令行构建，所有诊断抑制器将在所有分析器和编译器诊断计算完成**之后**运行。对于 Visual Studio 中的实时分析，为了支持增量和部分分析场景的优化，诊断抑制器可能只会接收完整已报告诊断集的子集。
   2. 各抑制器之间的执行顺序：诊断抑制器的执行相互独立。一个抑制器报告的诊断抑制，不会影响传递给其他抑制器的已报告诊断输入集。每个诊断可能被多个抑制器程序化抑制，因此分析器驱动会对每个诊断合并所有已报告的抑制。命令行编译器会为每个被程序化抑制的诊断，针对每项抑制记录一条抑制诊断。

# 开发体验示例

让我们来看 https://github.com/dotnet/roslyn/issues/30172 中 UnityEngine 的核心示例：

```csharp
using UnityEngine;

class BulletMovement : MonoBehaviour
{
    [SerializeField] private float speed;

    private void Update()
    {
         this.transform.position *= speed;
    }
}
```

嵌入 .NET Core 或 Mono 的序列化框架和工具，往往会在用户代码之外为字段赋值或调用方法。此类代码对编译器和分析器不可见，导致如下误报：

1. [CS0649](https://docs.microsoft.com/en-us/dotnet/csharp/misc/cs0649)：`字段"speed"从未被赋值，其值将始终为默认值 0`
2. [IDE0044 dotnet_style_readonly_field](https://docs.microsoft.com/en-us/visualstudio/ide/editorconfig-code-style-settings-reference?view=vs-2019)：将字段"speed"设为只读

## 附截图的新体验：

1. _面向平台/库作者：_ UnityEngine 可以通过基于 DiagnosticSuppressor API 编写简单的 [SerializedFieldDiagnosticSuppressor](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-serializedfielddiagnosticsuppressor-cs) 扩展，有选择地抑制此类诊断实例，并将其与 Unity 框架 SDK/库一起打包。
2. _面向最终用户：_
   1. 用户在 IDE 实时分析或命令行构建中不会看到上述误报。例如，查看以下有无 SerializedFieldDiagnosticSuppressor 时的截图对比：
      1. 没有抑制器：参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-without_suppressor-png)的图片
      2. 有抑制器：参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-with_suppressor-png)的图片
   2. 审计抑制：
      1. 命令行构建：最终用户会在命令行构建中看到"Info"严重性的抑制诊断，参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-suppression_info_diagnostic-png)的图片
      `info SPR0001: Diagnostic 'CS0649' was programmatically suppressed by a DiagnosticSuppressor with suppresion ID 'SPR1001' and justification 'Field symbols marked with SerializedFieldAttribute are implicitly assigned by UnityEngine at runtime and hence do not have its default value'`
      这些诊断将**始终**记录在 [msbuild binlog](https://github.com/Microsoft/msbuild/blob/master/documentation/wiki/Binary-Log.md) 中。在常规 msbuild 调用的控制台输出中不会显示，但将 msbuild 详细程度提升到 detailed 或 diagnostic 级别后，这些诊断将会输出。
      2. Visual Studio IDE：用户可在 Visual Studio 错误列表中更改"抑制状态"列的默认筛选器以包含"已抑制"的诊断，从而查看原始的已抑制诊断，参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-view_suppressed_diagnostic_errorlist-png)的图片。未来，我们还计划添加一个新的错误列表列（可能命名为"抑制来源"），用于显示与每项程序化抑制关联的抑制 ID 和理由。
   3. 禁用抑制器：最终用户可通过以下两种机制禁用分析器包中的特定抑制 ID：
      1. `/nowarn:<%suppression_id%>`：参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-disable_suppressor_nowarn-png)的图片，通过项目属性页禁用抑制 ID，这会生成 nowarn 命令行参数。
      2. 规则集条目：参见[此处](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#file-disable_suppressor_rulesetentry-png)的图片，使用 [CodeAnalysis 规则集](https://docs.microsoft.com/en-us/visualstudio/code-quality/using-rule-sets-to-group-code-analysis-rules?view=vs-2019)禁用抑制 ID。

# 待解问题

1. _DiagnosticSuppressor 功能应设为选择启用还是默认开启：_ 这一问题在过去被多次提出，尤其是对**默认开启**方案的顾虑——最终用户在命令行中不会看到任何表明诊断抑制器已作为构建一部分执行的提示。目前有以下选项：
   1. 选择启用：
      1. 在新的永久功能标志后保留该功能：这样命令行参数将始终说明诊断抑制器是否参与了构建。
      2. 新增命令行开关以启用抑制器。
   2. 默认开启：命令行中没有表明抑制器参与构建的参数，但用户可通过 binlog 和/或详细 msbuild 输出日志来确认。

   该功能的[草案 PR](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#draft-pr-implementing-the-above-proposal) 目前将该功能置于功能标志后。如果决定采用**默认开启**方案，可以还原添加功能标志的更改；如果决定通过新的命令行编译器开关使其选择启用，建议在初始 PR 中保留功能标志，然后通过后续 PR 添加命令行开关。

   **更新：** 已决定在初始版本中将该功能保留在临时功能标志后，待对功能的性能、用户体验等方面充分验证后再移除。

2. _"Info"抑制诊断的消息格式_：当前选用以下消息格式（含 3 个格式参数），欢迎提出修改建议：
   `Diagnostic 'CS0649' was programmatically suppressed by a DiagnosticSuppressor with suppresion ID 'SPR1001' and justification 'Field symbols marked with SerializedFieldAttribute are implicitly assigned by UnityEngine at runtime and hence do not have its default value'`

3. _"Info"抑制诊断的诊断 ID 应该是什么？_ 有以下选项：
   1. 使用 CSxxxx/BCxxxx 诊断 ID 作为抑制诊断，以明确表示该诊断来自核心编译器。
   2. 使用独特的诊断前缀，例如"SPR0001"，类似于我们报告分析器异常时使用的"AD0001"诊断方式。这为这些特殊的抑制诊断提供了独立的分类方式。
   该功能的[草案 PR](https://gist.github.com/mavasani/fcac17a9581b5c54cef8a689eeec954a#draft-pr-implementing-the-above-proposal) 选择了第二种方案，使用"SPR0001"作为诊断 ID。

   **更新：** 已决定诊断 ID 为 `SP0001`。

# 实现上述提案的草案 PR

https://github.com/dotnet/roslyn/pull/36067

# 草案 PR 中带有文档注释的详细 API 提案

```csharp
namespace Microsoft.CodeAnalysis.Diagnostics
{
    /// <summary>
    /// The base type for diagnostic suppressors that can programmatically suppress analyzer and/or compiler non-error diagnostics.
    /// </summary>
    public abstract class DiagnosticSuppressor : DiagnosticAnalyzer
    {
        // Disallow suppressors from reporting diagnostics or registering analysis actions.
        public sealed override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => ImmutableArray<DiagnosticDescriptor>.Empty;

        public sealed override void Initialize(AnalysisContext context) { }

        /// <summary>
        /// Returns a set of descriptors for the suppressions that this suppressor is capable of producing.
        /// </summary>
        public abstract ImmutableArray<SuppressionDescriptor> SupportedSuppressions { get; }

        /// <summary>
        /// Suppress analyzer and/or compiler non-error diagnostics reported for the compilation.
        /// This may be a subset of the full set of reported diagnostics, as an optimization for
        /// supporting incremental and partial analysis scenarios.
        /// A diagnostic is considered suppressible by a DiagnosticSuppressor if *all* of the following conditions are met:
        ///     1. Diagnostic is not already suppressed in source via pragma/suppress message attribute.
        ///     2. Diagnostic's <see cref="Diagnostic.DefaultSeverity"/> is not <see cref="DiagnosticSeverity.Error"/>.
        ///     3. Diagnostic is not tagged with <see cref="WellKnownDiagnosticTags.NotConfigurable"/> custom tag.
        /// </summary>
        public abstract void ReportSuppressions(SuppressionAnalysisContext context);
    }

    /// <summary>
    /// Provides a description about a programmatic suppression of a <see cref="Diagnostic"/> by a <see cref="DiagnosticSuppressor"/>.
    /// </summary>
    public sealed class SuppressionDescriptor : IEquatable<SuppressionDescriptor>
    {
        /// <summary>
        /// An unique identifier for the suppression.
        /// </summary>
        public string Id { get; }

        /// <summary>
        /// Identifier of the suppressed diagnostic, i.e. <see cref="Diagnostic.Id"/>.
        /// </summary>
        public string SuppressedDiagnosticId { get; }

        /// <summary>
        /// A localizable description about the suppression.
        /// </summary>
        public LocalizableString Description { get; }
    }

    /// <summary>
    /// Context for suppressing analyzer and/or compiler non-error diagnostics reported for the compilation.
    /// </summary>
    public struct SuppressionAnalysisContext
    {
        /// <summary>
        /// Suppressible analyzer and/or compiler non-error diagnostics reported for the compilation.
        /// This may be a subset of the full set of reported diagnostics, as an optimization for
        /// supporting incremental and partial analysis scenarios.
        /// A diagnostic is considered suppressible by a DiagnosticSuppressor if *all* of the following conditions are met:
        ///     1. Diagnostic is not already suppressed in source via pragma/suppress message attribute.
        ///     2. Diagnostic's <see cref="Diagnostic.DefaultSeverity"/> is not <see cref="DiagnosticSeverity.Error"/>.
        ///     3. Diagnostic is not tagged with <see cref="WellKnownDiagnosticTags.NotConfigurable"/> custom tag.
        /// </summary>
        public ImmutableArray<Diagnostic> ReportedDiagnostics { get; }

        /// <summary>
        /// Report a <see cref="Suppression"/> for a reported diagnostic.
        /// </summary>
        public void ReportSuppression(Suppression suppression);

        /// <summary>
        /// Gets a <see cref="SemanticModel"/> for the given <see cref="SyntaxTree"/>, which is shared across all analyzers.
        /// </summary>
        public SemanticModel GetSemanticModel(SyntaxTree syntaxTree);

        /// <summary>
        /// <see cref="CodeAnalysis.Compilation"/> for the context.
        /// </summary>
        public Compilation Compilation { get; }

        /// <summary>
        /// Options specified for the analysis.
        /// </summary>
        public AnalyzerOptions Options { get; }

        /// <summary>
        /// Token to check for requested cancellation of the analysis.
        /// </summary>
        public CancellationToken CancellationToken { get; }
    }

    /// <summary>
    /// Programmatic suppression of a <see cref="Diagnostic"/> by a <see cref="DiagnosticSuppressor"/>.
    /// </summary>
    public struct Suppression
    {
        /// <summary>
        /// Creates a suppression of a <see cref="Diagnostic"/> with the given <see cref="SuppressionDescriptor"/>.
        /// </summary>
        /// <param name="descriptor">
        /// Descriptor for the suppression, which must be from <see cref="DiagnosticSuppressor.SupportedSuppressions"/>
        /// for the <see cref="DiagnosticSuppressor"/> creating this suppression.
        /// </param>
        /// <param name="suppressedDiagnostic">
        /// <see cref="Diagnostic"/> to be suppressed, which must be from <see cref="SuppressionAnalysisContext.ReportedDiagnostics"/>
        /// for the suppression context in which this suppression is being created.</param>
        public static Suppression Create(SuppressionDescriptor descriptor, Diagnostic suppressedDiagnostic);

        /// <summary>
        /// Descriptor for this suppression.
        /// </summary>
        public SuppressionDescriptor Descriptor { get; }

        /// <summary>
        /// Diagnostic suppressed by this suppression.
        /// </summary>
        public Diagnostic SuppressedDiagnostic { get; }
    }
}
```
