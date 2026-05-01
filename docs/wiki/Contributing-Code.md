### 贡献代码

在提交功能或重大代码贡献之前，请先与团队讨论，并确保其符合产品[路线图](Roadmap.md)。团队会严格审查和测试所有代码提交。提交内容必须在质量、设计和路线图适宜性方面达到极高的标准。

Roslyn 项目是 [.NET Foundation](https://github.com/orgs/dotnet) 的成员，遵循相同的[开发者指南](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md)。团队通过定期在代码库上运行 [.NET 代码格式化工具](https://github.com/dotnet/codeformatter)来强制执行此规范。贡献者在提交时应确保遵循这些准则。

目前，团队对拉取请求设定了以下限制：

- 超出 bug 修复范围的贡献必须先与团队讨论，否则可能会被拒绝。随着我们流程的成熟和经验的积累，团队预计将接受更大规模的贡献。
- 只接受针对 main 分支的贡献。提交的拉取请求若以实验性功能分支或发布分支为目标，可能会被要求改为针对 main 分支。
- 与 main 分支最新提交无法顺利合并的拉取请求将被拒绝。作者将被要求与最新提交合并并更新拉取请求。
- 提交内容必须满足功能和性能预期，包括团队尚未开源测试的场景。这意味着如果你的拉取请求未能通过某些测试，可能会被要求修复并针对新的开放测试用例重新提交。
- 提交内容必须遵循每个目录的 [.editorconfig](http://editorconfig.org/) 设置。这些设置大体遵循 [.NET Foundation 编码准则](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md)中规定的规则，但大多数 Roslyn 项目倾向于在所有地方使用 'var'。
- 贡献者必须签署 [.NET CLA](https://cla.dotnetfoundation.org/)

当你准备好进行更改时，请先[构建](Building-Testing-and-Debugging.md)代码，并熟悉我们的工作流程和编码规范。以下两篇关于为开源项目贡献代码的博文也值得参考：Miguel de Icaza 的《开源贡献礼仪》和 Ilya Grigorik 的《不要"推送"你的拉取请求》。

在提交拉取请求之前，你必须签署[贡献者许可协议（CLA）](http://cla.dotnetfoundation.org)。要完成 CLA，请通过表单提交请求，并在收到包含文档链接的邮件后以电子方式签署 CLA。你只需完成一次 CLA，即可涵盖所有 .NET Foundation 项目。

### 开发者工作流程

1. 在分类过程中将工作项分配给开发者
2. Roslyn 和外部贡献者都应在本地 fork 中完成工作，并通过拉取请求提交代码供审阅。
3. 当拉取请求流程认为更改已准备好时，将直接合并到主干。

### 开始在 Visual Studio 中编写代码

请参阅我们的入门指南 [此处](https://github.com/dotnet/roslyn/blob/main/docs/contributing/Building%2C%20Debugging%2C%20and%20Testing%20on%20Windows.md)。

### 创建新 Issues

在问题跟踪器中创建新 issue 时，请遵循以下准则：

- 使用能够识别所述问题或所请求功能的描述性标题。例如，在描述编译器行为不符合预期的问题时，请以编译器应有的行为而非实际行为来撰写 bug 标题——"当在 Abcd 中使用 Xyz 时，C# 编译器应报告 CS1234。"
- 除影响程度外，不要设置其他 bug 字段。
- 详细描述问题或所请求的功能。
- 对于 bug 报告，还请：
    - 描述预期行为和实际行为。如果原因不明显（如崩溃情况），请解释为什么预期该行为是正确的。
    - 提供能重现问题的示例代码。
    - 指明所有相关的异常消息和堆栈跟踪。
- 订阅所创建 issue 的通知，以便接收后续问题的回复。

### 编码规范

- 使用 [.NET Foundation 编码准则](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md)中概述的编码风格
- 在公共边界处使用简单代码验证参数。不要使用 Contracts 或魔法辅助方法。

    ```csharp
    if (argument == null)
    {
        throw new ArgumentNullException(nameof(argument));
    }
    ```

- 对于不需要在零售版本中进行的检查，使用 `Debug.Assert()`。始终在断言中包含"message"字符串以标识失败条件。添加断言以记录对非局部程序状态或参数值的假设，例如"在解析的这一阶段，调用者应已将扫描器推进到 '.' token"。
- 避免在编译器热点路径中进行分配：
    - 避免使用 LINQ。
    - 避免在没有结构体枚举器的集合上使用 foreach。
    - 考虑使用对象池。编译器中有许多对象池的使用示例可供参考。

### 代码格式化工具

Roslyn 团队定期使用 [.NET 代码格式化工具](https://github.com/dotnet/codeformatter)，以确保代码库随时间保持一致的风格。我们传递给该工具的具体选项如下：

- `/nounicode`：一般情况下，我们遵循不在字符串字面量中嵌入 Unicode 字符的规则。但也有少数需要验证编译器行为的情况，因此该选项暂时禁用。
- `/copyright`：默认版权为 MIT。Roslyn 以 Apache2 协议发布，因此需要覆盖此选项。

### Visual Basic 规范与规则

对于所有在 Visual Basic 中有对应内容的 C# 准则，团队将准则的精神应用于 Visual Basic。有关空格、缩进、参数名称以及命名参数使用的准则通常也适用于 Visual Basic。`Dim` 语句也应遵循 C# 中 `var` 使用的准则。Visual Basic 特有的规则是：字段名称应以 `m_` 或 `_` 开头。团队倾向于将所有字段声明置于类型定义的开头。Visual Studio 成员下拉列表不显示 VB 中的字段。将字段置于类型开头有助于导航。

IDE 功能通常应同时支持 C# 和 VB。以下情况除外：

1. 该功能没有合适的 VB 对应内容。例如，'patterns' 是仅限 C# 的特性，因此围绕 patterns 的特定功能通常不需要相应的 VB 工作。
2. 该功能同时为 VB 实现的代价过高。在这种情况下，请询问团队是否可以不做 VB 版本，再做决定。但总体而言，将功能同时为 C# 和 VB 编写通常只比仅为单一语言编写略贵一点（尤其是在一开始就考虑多语言情况的前提下），因此通常应该这样做。

在创建同时适用于 C# 和 VB 的 IDE 功能时，应尽量共享代码。已有大量示例和现有组件可以做到这一点。如需帮助，请向团队寻求建议。

### 小技巧

我们的团队在开发时发现 [Roslyn 的增强源码视图](http://sourceroslyn.io/)很有帮助。

许多团队成员可以在 <https://gitter.im/dotnet/roslyn>、<https://gitter.im/dotnet/csharplang> 和 <https://discord.gg/csharp>（#roslyn）找到。
