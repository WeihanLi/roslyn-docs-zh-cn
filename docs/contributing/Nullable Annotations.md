# 可空注解

本文档描述了在 Roslyn 代码库中应如何处理可空注解。默认情况下，请遵循与 [dotnet/runtime](https://github.com/dotnet/runtime) 仓库相同的[指南](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/api-guidelines/nullability.md)。本文档旨在详细说明 Roslyn 中与该指南不同的地方，并重申经常出现的规则。

## 注解指南

- **应该**注解所有新代码
- **应该**在方法体已确保值不可能为 `null` 但编译器无法追踪的情况下使用 null 抑制运算符。示例：
```cs
struct SyntaxTrivia {
    void Example() {    
        if (this.HasStructure) {
            Use(this.GetStructure()!);
        }
    }
}
```
- **应该**使用显式验证（如 `Debug.Assert` 或 `Contract.ThrowIfNull`）来捕获程序内部不变量。
```cs
void M1(SyntaxNode node) {
    Debug.Assert(node.Parent is object);
    ...
    M2(node.Parent);
}
```
- **避免**在同一次提交中同时进行注解和重构代码。这会显著增加审查成本。建议将注解和重大重构分离到不同的提交中。小型修改和代码风格现代化（如将 `default(SyntaxToken)` 改为 `default`）是可以接受的。
- **不必**关注对象被释放或以其他方式进入无效状态后的注解有效性。例如，在池化对象被释放后操作该对象。这违反了对象的约定，注解在此处没有意义。
- **不必**关注在调用 `MoveNext` 之前或 `MoveNext` 返回 false 之后使用枚举器 `Current` 的注解有效性。与 `Dispose` 一样，这种使用被认为是无效的，我们不希望为了迁就错误使用而损害 `Current` 的正确使用。
- **应该**在定义只为提供方法的非空返回版本（该方法返回可空引用类型）的成员时使用形容词 `Require`。例如：`GetRequiredSemanticModel` 从 `GetSemanticModel` 返回非空值。

## 破坏性变更指南

Roslyn 是一个大型代码库，我们需要相当长的时间才能完全注解整个代码库。对于我们较大的库，这可能需要数个发布版本才能完成。在此期间，这些库不会将可空注解视为兼容性标准的一部分。也就是说，可空注解的变更不需要在破坏性变更文档中记录，也不需要进行兼容性审查等。

一旦某个库的可空注解达到临界质量，我们将开始将可空注解作为兼容性标准的一部分进行跟踪。Roslyn 团队仍然保留在语言或底层平台实际需要时更改注解的权利。但此类变更将受到更严格的审查，并需要 API 兼容性委员会成员的批准。

当特定库完成其可空注解后，这些注解将被[跟踪在 API 文件中](https://github.com/dotnet/roslyn-analyzers/pull/3125)，库名称将添加到本文档中作为已完成的标记。
