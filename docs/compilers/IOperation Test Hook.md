# IOperation 测试钩子

编译器内置了一个测试钩子，旨在确保 IOperation 节点和场景与语义模型中可检索到的信息保持一致，并帮助确保我们的代码覆盖率足够，使得 IOperation 在我们没有直接测试的场景中不会出现失败或违反不变量的情况。其工作原理是：每当测试框架创建 `Compilation` 对象时，就会运行此钩子——对于编译中的每棵语法树，我们枚举每个语法节点，并验证 `GetOperation` 不会崩溃，且其返回信息与 `SemanticModel` 保持一致（例如，`GetTypeInfo` 与 `IOperation` 类型匹配）。此外，我们还会获取并验证编译中每个成员体的控制流图，并运行 CFG 验证器以确保所有不变量得以保持。

## 特性开发

在特性开发过程中，当我们调整 `BoundNode`、引入新节点或启用之前不存在的代码时，钩子可能会触发失败——原因是 IOperation 机制尚未处理特定节点，或者其逻辑因新代码违反了某些已有假设而触发了 `Debug.Assert`。修复此类问题有以下几种方式：

1. 在同一 PR 中实现 IOperation 的相应更改。这是在生产分支中进行此操作的唯一可接受方式。
2. 将新的 `BoundKind` 添加到 `CSharpOperationFactory`/`VisualBasicOperationFactory.Create` 底部的通用 switch 分支，并添加原型注释，确保在特性分支合并回生产分支之前完成支持实现。有时测试会命中其他失败点，例如 `SemanticModel` 失败或不匹配。对于这类情况，可以用 `ConditionalFact(typeof(NoIOperationValidation))` 标记测试以在测试钩子中跳过。执行此操作时，请添加原型注释（可以作为上方的注释，也可以作为 `ConditionalFactAttribute` 的 reason 字符串），以确保在合并到生产分支之前修复该问题。

## 复现失败

要复现测试失败，有以下两种方式：

1. 取消注释 `src/Compilers/Test/Core/Compilation/CompilationExtensions.cs:7`（该行定义了 `ROSLYN_TEST_IOPERATION`），然后运行测试。请**不要**将此更改提交，因为这会为每个项目的每个测试启用测试钩子，导致常规测试运行明显变慢。
2. 设置 `ROSLYN_TEST_IOPERATION` 环境变量，并在设置后重启 VS。
3. 在 `src/Compilers/Test/Core/Compilation/CompilationExtensions.cs` 中的 `ValidateIOperations` 开头设置断点。当断点触发时，使用 VS 的"跳转到位置"功能，或将指令指针拖过 `EnableVerifyIOperation` 处的提前检查和返回，从而为当前测试运行执行该代码。

完成上述任一操作后，所有测试都会在运行过程中执行钩子，你可以正常设置断点。通常有帮助的做法是运行直到异常触发，然后查看正在处理的节点，因为大多数测试包含多个方法，了解具体上下文有助于找出可能出错的早期步骤。

当测试失败被隔离后，请为此添加一个**专用的** IOperation 测试，以便更容易避免未来的回归。最好不要复制整个原始测试，只需保留足够触发 bug 的部分，以确保其受到回归保护。
