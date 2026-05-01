# `field` 关键字可空性分析实现

另请参见[规范](https://github.com/dotnet/csharplang/blob/94205582d0f5c73e5765cb5888311c2f14890b95/proposals/field-keyword.md#nullability)。

## Symbol API 行为

为了确定 SynthesizedBackingFieldSymbol 的 NullableAnnotation，我们需要：
1. 判断与此字段关联的 getter 是否需要空值弹性分析。
2. 如果*不需要*此类分析，则使用属性的可空注解。
3. 如果*需要*，则必须对 getter 执行绑定+可空分析，以确定后备字段的可空性。

直接在此字段符号上公开可空注解存在一些重大问题。
1. 性能成本。尚不清楚读取字段的 NullableAnnotation 是否会导致绑定和可空分析是否可接受。如果某些工具正在遍历成员符号进行索引等操作，若此举导致之前未发生的意外方法绑定，可能会造成问题。
2. 栈循环。在绑定+流分析过程中，我们可能需要访问字段的可空注解。如果我们*已经*在确定该字段的可空注解，则需要"短路"并返回某个"占位符"值。我们需要注意将"内部"实现与任何公共实现隔离，以确保向公共 API 用户给出一致的答案。
...

避免在公共符号模型中公开推断的可空注解很有吸引力。但对于试图推断可空初始化的自动化工具来说，这可能是个问题。例如，如果诊断抑制器想要在某些条件下抑制 `CS8618` 可空初始化警告。它应该如何决定像 `string Prop => field ??= GetValue();` 这样的属性是否需要在构造函数中初始化？目前尚不清楚这是否重要。

为了*简单性*（先让它正确，再让它快速），我希望通过*不*在符号模型中公开推断的注解来推进。相反，我们引入一个新的内部 API `SynthesizedBackingFieldSymbol.GetInferredNullableAnnotation()`，`NullableWalker` 将使用它来决定初始可空状态、报告警告等。该实现只需对 get 访问器进行按需绑定，然后进行可空分析并决定可空注解。需要注意避免以可重入方式使用此 API——因此，用于推断可空注解的 `NullableWalker` 遍历必须避免调用它。

这意味着普通的 `FieldSymbol.TypeWithAnnotations` API 不会公开推断的可空性，`IFieldSymbol.Type` 或 `IFieldSymbol.NullableAnnotation` 也不会。相反，字段的可空性将始终与属性相匹配。这是我实际上想要修复的问题，也许在合并到 main 之前。感觉快速信息等应该通过普通 API 公开推断的可空性。

一旦我们掌握了 getter 空值弹性的行为，我们就可以开始讨论如何降低与之相关的成本。例如，我们可以通过在分析某些方法之前强制推断可空注解来减少冗余绑定。例如，在 `MethodCompiler` 中处理构造函数之前，也许我们可以识别出需要为推断目的而编译的 getter，并强制这些 getter 先编译。
