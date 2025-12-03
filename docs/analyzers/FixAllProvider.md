# 简介

本文档涵盖以下内容：

- 入门定义
- 什么是 FixAll 出现位置代码修复？
- 如何计算 FixAll 出现位置代码修复？
- 为代码修复器添加 FixAll 支持
- 为代码操作选择等效键
- FixAll 提供程序的范围
- 内置 FixAllProvider 及其限制
- 实现自定义 FixAllProvider

# 定义

- **分析器：** 从 `DiagnosticAnalyzer` 派生的类型的实例，用于报告诊断。
- **代码修复器：** 从 `CodeFixProvider` 派生的类型的实例，为编译器和/或分析器诊断提供代码修复。
- **代码重构：** 从 `CodeRefactoringProvider` 派生的类型的实例，提供源代码重构。
- **代码操作：** 由 `CodeFixProvider.RegisterCodeFixesAsync` 注册的执行代码修复的操作，或由 `CodeRefactoringProvider.ComputeRefactoringsAsync` 注册的执行代码重构的操作。
- **等效键：** 表示由代码修复器或重构注册的所有代码操作的等效类的字符串值。如果两个代码操作具有相等的 `EquivalenceKey` 值并且由同一代码修复器或重构生成，则它们被视为等效。
- **FixAll 提供程序：** 从 `FixAllProvider` 派生的类型的实例，提供 FixAll 出现位置代码修复。FixAll 提供程序通过 `CodeFixProvider.GetFixAllProvider` 方法与相应的代码修复器关联。
- **FixAll 出现位置代码修复：** 由 `FixAllProvider.GetFixAsync` 返回的代码操作，修复相应代码修复器在给定 `FixAllScope` 内修复的所有或多个诊断出现位置。

# 什么是 FixAll 出现位置代码修复？

通俗地说，FixAll 出现位置代码修复意味着：我有一个代码修复 'C'，它修复我的源代码中诊断 'D' 的特定实例，我想将此修复应用于更广范围内的所有 'D' 实例，例如文档、项目或整个解决方案。

更技术性地说：给定代码修复器注册的用于修复一个或多个诊断的特定代码操作，其 FixAll 提供程序注册的相应代码操作跨更广范围（如文档/项目/解决方案）应用原始触发代码操作，以修复此类诊断的多个实例。

# 如何计算 FixAll 出现位置代码修复？

以下步骤用于计算 FixAll 出现位置代码修复：

- 给定诊断的特定实例，计算声称修复该诊断的代码操作集。
- 从此集合中选择特定的代码操作。在 Visual Studio IDE 中，这是通过在灯泡菜单中选择特定代码操作来完成的。
- 所选代码操作的等效键表示必须作为 FixAll 出现位置代码修复的一部分应用的代码操作类。
- 给定此代码操作，获取与注册此代码操作的代码修复器相对应的 FixAll 提供程序。
- 如果非空，则请求 FixAll 提供程序获取其支持的 FixAllScopes。
- 从此集合中选择特定的 `FixAllScope`。在 Visual Studio IDE 中，这是通过单击预览对话框中的范围超链接来完成的。
- 给定触发诊断、触发代码操作的等效键和 FixAll 范围，调用 `FixAllProvider.GetFixAsync` 来计算 FixAll 出现位置代码修复。

# 为代码修复器添加 FixAll 支持

请按照以下步骤为代码修复器添加 FixAll 支持：

- 重写 `CodeFixProvider.GetFixAllProvider` 方法并返回一个非空的 `FixAllProvider` 实例。您可以使用我们的内置 FixAllProvider 或实现自定义 FixAllProvider。请参阅本文档中的以下部分，以确定适合您的修复器的正确方法。
- 确保代码修复器注册的所有代码操作都有非空的等效键。请参阅以下部分以确定如何选择等效键。

# 为代码操作选择等效键

代码修复器的每个唯一等效键定义一个唯一的代码操作等效类。触发代码操作的等效键是 `FixAllContext` 的一部分，用于确定 FixAll 出现位置代码修复。
通常，您可以使用代码操作的 **'title'** 作为等效键。但是，在某些情况下，您可能希望使用不同的值。让我们通过一个例子来更好地理解。

让我们考虑 [C# SimplifyTypeNamesCodeFixProvider](https://github.com/dotnet/roslyn/blob/main/src/Features/CSharp/Portable/SimplifyTypeNames/SimplifyTypeNamesCodeFixProvider.cs)，它注册多个代码操作并且也有 FixAll 支持。此代码修复器提供修复以简化以下表达式：

- 形如 'this.x' 的 `this` 表达式简化为 'x'。
- 形如 'A.B' 的限定类型名简化为 'B'。
- 形如 'A.M' 的成员访问表达式简化为 'M'。

此修复器需要相应的 FixAll 出现位置代码修复具有以下语义：

- `this` 表达式简化：Fix all 应简化所有 this 表达式，无论访问的成员是什么（this.x、this.y、this.z 等）。
- 限定类型名简化：Fix all 应将所有限定类型名 'A.B' 简化为 'B'。但是，我们不想简化**所有**限定类型名，如 'C.D'、'E.F' 等，因为那将是一个过于通用的修复，用户可能并不希望这样。
- 成员访问表达式：Fix all 应将所有成员访问表达式 'A.M' 简化为 'M'。

它使用以下等效键为其注册的代码操作获取所需的 FixAll 行为：

- `this` 表达式简化：通用资源字符串 "Simplify this expression"，明确排除正在简化的节点内容。
- 限定类型名简化：格式化资源字符串 "Simplify type name A.B"，明确包含正在简化的节点内容。
- 成员访问表达式：格式化资源字符串 "Simplify type name A.M"，明确包含正在简化的节点内容。

请注意，'`this` 表达式简化'修复需要与其他两种简化不同类型的等效类。请参阅方法 [GetCodeActionId](https://github.com/dotnet/roslyn/blob/main/src/Analyzers/Core/CodeFixes/ImplementAbstractClass/AbstractImplementAbstractClassCodeFixProvider.cs) 了解实际实现。

总之，使用最适合作为 FixAll 操作一部分应用的修复类别的等效键。

# FixAll 提供程序的范围

当需要将多个修复应用于文档时，有多种方法：

- **顺序方法：** 一种方法是计算诊断，选择一个，请求修复器生成代码操作来修复它，应用它。现在对于生成的新编译，重新计算诊断，选择下一个并重复该过程。这种方法会非常慢，但会产生正确的结果（除非它不收敛，即一个修复引入了刚刚被前一个修复修复的诊断）。我们选择不实现此方法。
- **批量修复方法：** 另一种方法是计算所有诊断，选择每个诊断并将其提供给修复器并应用它以生成新的解决方案。如果有 'n' 个诊断，将有 'n' 个新解决方案。现在只需一次性将它们全部合并在一起。这可能会产生不正确的结果（当不同的修复以不同的方式更改相同的代码区域时），但它非常快。我们在 `WellKnownFixAllProviders.BatchFixer` 中有此方法的一个实现
- **自定义方法：** 根据修复，可能有自定义解决方案来修复多个问题。例如，考虑一个分析器只需要生成一个文件作为任何问题实例的修复。与使用前两种方法反复生成相同的文件不同，可以编写一个自定义 `FixAllProvider`，只要有任何诊断就简单地生成一次文件。

由于有多种修复所有问题的方法，我们实现了一个框架并提供了一个我们认为在许多情况下有用的通用实现。

# 内置 FixAllProvider

我们提供了 FixAll 提供程序的默认 `BatchFixAllProvider` 实现，它使用底层代码修复器来计算 FixAll 出现位置代码修复。
要使用批量修复器，您应该在 `CodeFixProvider.GetFixAllProvider` 重写中返回静态 `WellKnownFixAllProviders.BatchFixer` 实例。
注意：请参阅以下关于 **'BatchFixer 的限制'** 的部分，以确定批量修复器是否可以被您的代码修复器使用。

给定触发诊断、触发代码操作、底层代码修复器和 FixAll 范围，BatchFixer 通过以下步骤计算 FixAll 出现位置代码修复：

- 计算 FixAll 范围内触发诊断的所有实例。
- 对于每个计算的诊断，调用底层代码修复器来计算修复诊断的代码操作集。
- 收集所有具有与触发代码操作相同等效键的已注册代码操作。
- 在原始解决方案快照上应用所有这些代码操作以计算新的解决方案快照。批量修复器只批处理各个代码操作中存在的 `ApplyChangesOperation` 类型的代码操作操作，其他类型的操作将被忽略。
- 顺序地将新的解决方案快照合并到最终的解决方案快照中。只保留修复范围不与先前合并的代码操作的修复范围重叠的非冲突代码操作。

# BatchFixer 的限制

BatchFixer 是为诊断的修复范围彼此不重叠的常见修复器类别设计的。例如，假设有一个跨越特定表达式的诊断，以及一个修复该表达式的修复器。如果此诊断的所有实例都保证具有不重叠的范围，则可以独立计算它们的修复，并且可以随后将这批修复合并在一起。

但是，在某些情况下，BatchFixer 可能不适用于您的修复器。以下是一些此类示例：

- 代码修复器注册没有等效键或具有空等效键的代码操作。
- 代码修复器注册非本地代码操作，即其修复范围与诊断范围完全不同的代码操作。例如，添加新声明节点的修复。多个此类修复可能具有重叠的范围，因此可能会冲突。
- 作为 FixAll 出现位置一部分要修复的诊断具有重叠的范围。此类诊断的修复也可能具有重叠的范围，因此会彼此冲突。
- 代码修复器注册具有 `ApplyChangesOperation` 以外操作的代码操作。BatchFixer 会忽略此类操作，因此可能会产生意外结果。

# 实现自定义 FixAllProvider

对于无法使用 BatchFixer 的情况，您必须实现自己的 `FixAllProvider`。建议您创建 FixAll 提供程序的单例实例，而不是为每次 `CodeFixProvider.GetFixAllProvider` 调用创建新实例。
以下指南应有助于实现：

- **GetFixAsync：** 为给定的 `FixAllContext` 计算 FixAll 出现位置代码修复的主要方法。您可以使用 `FixAllContext` 上的 'GetXXXDiagnosticsAsync' 方法集来计算要修复的诊断。您必须返回一个修复给定 FixAll 范围内所有诊断的单个代码操作。
- **GetSupportedFixAllScopes：** 获取所有支持的 FixAll 范围的虚方法。默认情况下，它返回所有三个支持的范围：文档、项目和解决方案范围。通常，您不需要重写此方法。但是，如果您希望支持这些范围的子集，可以这样做。
- **GetSupportedFixAllDiagnosticIds：** 获取所有可修复诊断 ID 的虚方法。默认情况下，它返回底层代码修复器的 `FixableDiagnosticIds`。通常，您不需要重写此方法。但是，如果您希望仅为这些 ID 的子集支持 FixAll，可以这样做。

请参阅 [DeclarePublicAPIFix](https://github.com/dotnet/roslyn/blob/main/src/RoslynAnalyzers/PublicApiAnalyzers/Core/CodeFixes/DeclarePublicApiFix.cs) 了解自定义 FixAllProvider 的示例实现。
