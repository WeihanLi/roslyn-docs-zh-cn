# 编译器调试/EnC 检查清单

1. 序列点正确生成，且对应的语法节点被 IDE 识别为断点跨度
    - 通过使用新编译器位启动 VS 并逐步执行，以及在所有应允许断点的语法节点上放置断点来手动验证
    - 添加回归编译器测试，检查带序列点的已生成 IL（`VerifyIL` 与 `sequencePoints`）
    - 通过 `src\EditorFeatures\CSharpTest\EditAndContinue\BreakpointSpansTests.cs` 添加 IDE 测试（实现在 `src\Features\CSharp\Portable\EditAndContinue\BreakpointSpans.cs`，处理语法到序列点的映射）
2. 序列点与闭包分配及其他隐藏代码的关系（针对任何产生序列点的语法）
    - 调试器通过"设置下一语句 Ctrl+Shift+F10"命令支持手动移动当前 IP（指令指针）。
    - 语句可以设置为当前方法中的任何序列点。
    - 需要确保序列点的生成方式使正确的隐藏代码得到执行。
    - 例如，分配闭包的代码块的左大括号上的序列点需要在闭包分配指令之前，这样当 IP 设置到此序列点时，闭包能被正确分配。
```
{               // 在此处分配闭包
   var x = 1;
   F(() => x);
}
```
3. 条件分支在 DEBUG 构建中必须使用 stloc/ldloc 模式
    - 检查调试构建中 if 语句等的指令和序列点生成。我们不使用简单的 brtrue/brfalse，而是分配一个临时局部变量，将条件评估结果存储到该局部变量，生成隐藏序列点，从局部变量加载结果然后分支。这支持 EnC 函数重映射。只需在实现生成包含可能含任意表达式的条件分支的功能时考虑这一点。我相信降低阶段中有一些通常生成条件的辅助方法，只要使用正确的辅助方法，这应该会自动完成。但这是需要注意的。
4. 闭包和 lambda 作用域（PDB 信息）
    - 如果引入新的语法表示某种 lambda（匿名方法、局部函数、LINQ 查询等），相应地更新 `src\Compilers\CSharp\Portable\Syntax\LambdaUtilities.cs` 中的辅助方法
    - 如果引入新的作用域，可以声明可提升到闭包中的变量
      - 表示作用域的绑定节点需要与 `src\Compilers\CSharp\Portable\Syntax\LambdaUtilities.cs` 中辅助方法识别的语法节点关联（特别是 `IsClosureScope`）。
      - 此要求由 `SynthesizedClosureEnvironment` 构造函数中的断言强制执行。
    - Lambda 和闭包语法偏移量必须生成到 PDB（encLambdaMap 自定义调试信息）
      - 闭包的偏移属性标识与闭包关联的语法节点。此偏移量必须唯一。
5. 引入新符号时，符号匹配器可能需要更新
    - 符号匹配器将一个编译中的符号映射到另一个编译。
    - 合成符号（如闭包、状态机、匿名类型、lambda 等）也必须被映射。
    - 实现：`src\Compilers\CSharp\Portable\Emitter\EditAndContinue\CSharpSymbolMatcher.cs`
    - 测试：`src\Compilers\CSharp\Test\Emit\Emit\EditAndContinue\SymbolMatcherTests.cs`
6. 引入可能声明用户局部变量或生成长期合成变量的新语法时（即在断点之间持续存在的状态，而不仅仅是表达式中的临时变量）
    - 验证变量槽可以从新编译映射到之前的编译
    - 这由 `EncVariableSlotAllocator` 使用存储在 PDB 中的语法偏移量实现（`encLocalSlotMap` 和 `encLambdaMap` 自定义调试信息）。
    - 当前机制可能不足以支持映射，在这种情况下，请与 IDE 团队讨论以设计额外的 PDB 信息来支持映射。
7. 每个新语言功能都应在 Emit 测试下的 Emit/EditAndContinue 中有相应测试。
    - 某些功能可能只需要一个测试，其他功能可能需要多个测试，具体取决于对 EnC 的影响。
    - PDB 测试验证这些作用域（在 PDB XML 中查找 `<scope>`）。`LocalsTests.cs` EE 测试也验证作用域。
8. 引入可能为局部变量声明作用域的新语法时，需要在 PDB 中正确生成对应的 IL 作用域
    - EE 使用这些来确定哪些变量在作用域内。
9. 测试该功能的调试体验
    - 监视窗口中是否显示有用信息？
    - 我可以在监视窗口中使用此功能评估表达式吗？
    - 某些功能可能需要添加更多自定义 PDB 信息以提供良好的调试体验（例如异步、迭代器、动态等）。
    - 与 IDE 团队一起设计改进体验和自定义 PDB 信息。
