# 可空性 API 设计说明

## 指导原则

在设计此 API 时，我们尝试牢记以下几个核心原则：

1. 首要原则是不破坏现有的分析器和重构工具。这意味着以前有效的常见模式需要继续有效，包括对非泛型 `ITypeSymbol`（例如 `string`）的引用相等比较。
2. 我们希望在不需要时能够避免计算可空性信息。目前，`SemanticModel` API 只进行回答问题所需的最少绑定量，但尝试计算可空性信息将导致更多内容被绑定。要完全确定可空性，必须绑定整个方法并运行可空性分析，而今天的单个语句可以单独绑定。即使标准路径计算可空性信息，我们认为某些对性能敏感的场景也不关心可空性分析结果，例如智能感知功能。
3. 用户应能够获取变量的声明可空性，以及查看流分析中每个表达式的推断可空性。

## 一般概念

可空性信息通过两个枚举公开：`NullableAnnotation` 和 `NullableFlowState`。

`NullableAnnotation` 表示左值，有 4 个可能的值：

* `NotApplicable` — 当对没有可空性概念的内容（例如语句、指令等）请求可空性信息时使用。
* `Disabled` — 该值在可空性禁用的上下文中定义。
* `NotAnnotated` — 该值在可空性启用的上下文中定义，且在源码中未标注（或在类型推断场景中推断为未标注类型）。
* `Annotated` — 该值在可空性启用的上下文中定义，且在源码中用 `?` 标注（或在类型推断场景中推断为标注类型）。

`NullableFlowState` 表示右值，有 3 个可能的值：

* `NotApplicable` — 同上，用于没有可空性概念的构造。
* `MaybeNull` — 编译器的可空分析确定该值可能为 null。
* `NotNull` — 编译器的可空分析确定该值不为 null。

可空性信息将从 `SemanticModel` 获取，类似于类型信息。`GetSemanticModel` 将返回一个感知可空性信息的 `SemanticModel`，查询时会强制绑定并计算可空性信息。我们可能提供 `GetSemanticModel(bool skipNullabilityInformation)` 重载，为性能敏感场景返回不计算可空性信息的 `SemanticModel`。该 `SemanticModel` 将与可空感知的 `SemanticModel` 共享缓存，从中获取的符号的所有可空性将为 `NotAnalyzed`。此 API 仅在 dogfooding 后确定某些场景性能足够差时才会添加。

通常，当需要确定 `ITypeSymbol` 的可空性时，需要检查包含上下文。声明符号（例如 `ILocalSymbol`、`IFieldSymbol` 等）将具有该类型的 `NullableAnnotation`。`GetTypeInfo` 将提供 `ITypeSymbol`、`NullableAnnotation`（如果此表达式可用作左值）以及表达式的 `NullableFlowState`（如果此表达式可用作右值）。类型参数的嵌套 `NullableAnnotation` 包含在包含 `ITypeSymbol` 上。例如，若要了解数组内容是否可为 null，需要检查 `IArrayTypeSymbol` 的 `ElementNullableAnnotation`。

值类型始终被认为是 `NotNull`。可空值类型和引用类型的状态会被跟踪，根据流状态为 `NotNull` 或 `MaybeNull`。无约束泛型参数也在跟踪范围内。使用 `GetSymbolInfo`/`GetDeclaredSymbol` 查询时，无约束类型参数的变量/参数/字段等具有 `NotAnnotated` 的 `NullableAnnotation`。使用 `GetTypeInfo` 查询时，如果尚未检查 null 且未直接或间接赋非 null 值，流状态为 `MaybeNull`；检查 null 后且未被赋 `MaybeNull` 值后，流状态为 `NotNull`。

### 声明可空性

声明可空性是程序员通常在源代码中显式键入的可空性，始终由 `NullableAnnotation` 枚举表示。某些情况下会被推断，例如 lambda 参数/返回类型、`var` 变量和类型参数替换。对于这些情况，需要流状态来计算声明可空性。例如：

```C#
string? s1 = string.Empty;
var s2 = s1;
s2 = null; // 这将产生警告，因为 s2 的类型是 string，而不是 string?
```

`s1` 的声明标注为 `Annotated`，即使它被初始化为非 null 值。但由于 `s1` 的流状态为 `NotNull`，我们推断 `s2` 的声明标注也为 `NotAnnotated`。

声明可空标注可以从声明符号（例如 `IFieldSymbol`、`IMethodSymbol` 等）获取。某个位置的符号可空标注可以通过 `GetSymbolInfo` API 获取：如果表达式成功解析为单个符号，`Symbol` 属性将具有计算好的标注。`CandidateSymbols` 不会有计算好的可空标注，因为我们不对重载解析失败的代码运行可空分析。

### 流状态

启用可空特性后，编译器将跟踪整个方法中表达式的流状态，无论变量声明为何种类型。流状态始终由 `NullableFlowState` 枚举表示。`SemanticModel` 可用于通过 `GetTypeInfo` API 请求单个表达式或子表达式的流状态信息。`TypeInfo` 结构将增加 `Nullability` 和 `ConvertedNullability` 信息，类似于目前拥有 `Type` 和 `ConvertedType`。

### `GetSemanticModel(bool skipNullabilityInformation)`

此 API 提案改变了 `SemanticModel` 调用的默认行为。目前，如果请求 `SyntaxNode` 的语义信息，我们尽量只绑定提供语义信息所需的最少内容，通常是单个语句。但如果该语义信息将包含 `Nullability`，必须绑定该语句之前的所有语句并运行可空分析直到并包括该语句。我们认为此更改对大多数场景是可接受的，因为着色和分析器等功能已经在后台近乎持续地运行，绑定屏幕上和文件中的几乎所有内容。对于少数性能敏感的场景，将提供跳过可空信息的 `GetSemanticModel()` 重载，它将与标准语义模型共享初始绑定的 `BoundNode` 缓存。

## 新 API

我们为获取和使用可空性添加了以下新 API：

```C#
public enum NullableAnnotation : byte
{
    None,
    NotAnnotated,
    Annotated
}

public enum NullableFlowState : byte
{
    NotApplicable,
    NotNull,
    MaybeNull
}

public readonly struct NullabilityInfo
{
    public NullableAnnotation Annotation { get; }
    public NullableFlowState FlowState { get; }
}

#region Declared Nullability

// 这些可能受计算流状态影响，但在初始声明后不再改变

public interface IDiscardSymbol : ISymbol
{
    NullableAnnotation NullableAnnotation { get; }
}

public interface IEventSymbol : ISymbol
{
    NullableAnnotation NullableAnnotation { get; }
}

public interface IFieldSymbol : ISymbol
{
    NullableAnnotation NullableAnnotation { get; }
}

public interface ILocalSymbol : ISymbol
{
    NullableAnnotation NullableAnnotation { get; }
}

public interface IMethodSymbol : ISymbol
{
    NullableAnnotation ReturnNullableAnnotation { get; }
    ImmutableArray<NullableAnnotation> TypeArgumentsNullableAnnotations { get; }
    NullableAnnotation ReceiverNullableAnnotation { get; }
}

public interface IParameterSymbol : ISymbol
{
    NullableAnnotation NullableAnnotation { get; }
}

public interface IArrayTypeSymbol : ITypeSymbol
{
    NullableAnnotation ElementNullableAnnotation { get; }
}

public interface INamedTypeSymbol : ITypeSymbol
{
    ImmutableArray<NullableAnnotation> TypeArgumentsNullableAnnotations { get; }
}

public interface ITypeParameterSymbol : ITypeSymbol
{
    NullableAnnotation ReferenceTypeConstraintNullableAnntotation { get; }
    ImmutableArray<NullableAnnotation> ConstraintsNullableAnnotations { get; }
}

#endregion

#region Flow State Nullability

public struct TypeInfo
{
    public NullabilityInfo Nullability { get; }
    public NullabilityInfo ConvertedNullability { get; }
}

#endregion

#region Compilation

public class Compilation
{
    // 仅在 dogfooding 后确定性能敏感场景需要时才添加此 API
    public SemanticModel GetSemanticModel(bool skipNullabilityInformation);
}

#endregion

#region Printing

public interface ITypeSymbol
{
    string ToDisplayString(NullableFlowState nullability, SymbolDisplayFormat format = null);
    ImmutableArray<SymbolDisplayPart> ToDisplayParts(NullableFlowState nullability, SymbolDisplayFormat format = null);
    string ToMinimalDisplayString(SemanticModel semanticModel, int position, NullableFlowState nullability, SymbolDisplayFormat format = null);
    ImmutableArray<SymbolDisplayPart> ToMinimalDisplayString(SemanticModel semanticModel, int position, NullableFlowState nullability, SymbolDisplayFormat format = null);
}

/*
 * 还将有更多 API 更改，例如 SyntaxFactory 和 SyntaxGenerator 上的。设计准备好后此节将更新。
 */

#endregion
```

## 备选方案

### `ITypeSymbol` 上的 `Nullability`

此备选提案的基本原则是向 `ITypeSymbol` 添加 `Nullability Nullability { get; }`，并删除主节中其余的可空性获取 API。优点包括：消费者无需追踪类型符号来源、打印和代码生成 API 无需新重载、消费者无需在比较类型时单独担心可空性。但此方法有几个主要问题：

1. 会违反 API 目标的原则 #1，大量破坏现有分析器和重构工具。通过 `Compilation.GetTypeByMetadataName` 或类似重载，然后查找请求类型的字段、局部变量、属性等是常见策略。以前对非泛型类型可以直接用 `==` 比较，但添加 `Nullability` 后这将不再可行。
2. 符号的默认 `Nullability` 是另一个痛点。声明类时没有可空性，类只是 `INamedTypeSymbol`。为了不破坏可空性未感知代码上的分析器，必须使默认可空性为 `Disabled`，这会加剧问题 #1。
3. 导致类型层次结构急剧膨胀。如果用户获取 `string?` 并调用 `GetDeclaredMembers()`，每个成员的父类必须指向 `string?` 而不是 `string` 或 `string~`。

因此，我们决定不采用此方法。

### `GetSingleStatementSemanticModel`

在 `SemanticModel` 上添加 `GetSingleStatementSemanticModel()` 方法，返回共享 `BoundNode` 缓存且不计算可空信息的 `SemanticModel`。该方案因过于复杂且缺乏前瞻性（若未来添加更多用户希望退出的分析）而被拒绝。

### `GetNullabilityAwareSemanticModel`

此备选提案与上一个相反：`SemanticModel` 默认不计算可空性，仅在请求时才计算。我们决定不采用此方案，因为我们认为除特定性能问题的场景外，不计算可空性的节省通常无效，IDE 特性或分析器已经请求可空性，节省的时间将被浪费。

### `GetNullabilityInfo`

早期备选提案有一个与 `GetTypeInfo()` 并行的 API `GetNullabilityInfo()`，返回包含可空信息的结构。我们决定此方案没有好处：由于嵌套可空性，调用 `GetTypeInfo` 时仍需运行可空分析，且会不必要地分离用户希望同时访问的信息。此外，我们无法为 `GetDeclaredSymbol` 和 `GetSymbolInfo` 提出配套的好 API。

### 延迟计算的 `Nullability` 属性

此备选提案向接口添加了与主提案类似的 `Nullability` 属性，但不预先计算，而是在用户请求时才计算。主要问题在于符号、`TypeInfo` 和 `SymbolInfo` 必须携带调用上下文才能延迟计算这些属性，这是一个重大的破坏性更改。

### `ITypeReference`

我们短暂考虑了添加 `ITypeReference` API 的思路，它包含 `ITypeSymbol` 和 `Nullability`，以及 ref、自定义修饰符等附加信息。这将是一个极其破坏性的更改，需要对整个 API 表面进行大量修改，且需要重新审视 Roslyn 初始设计中的一个明确设计选择。

### 左值和右值使用单一 `Nullability` 枚举

此提案与当前提案非常相似，只是只有一个公共枚举 `Nullability`，可能的值为 `NotApplicable`、`NotComputed`、`Unknown`、`MaybeNull` 和 `NotNull`。经过一些实现工作后，我们认为将右值状态和左值标注分开将产生更有用、更易维护和测试的 API。
