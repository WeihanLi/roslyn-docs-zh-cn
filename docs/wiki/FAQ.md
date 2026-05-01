# .NET 编译器平台（"Roslyn"）常见问题解答

本 FAQ 包含学习入门问题以及常见问题，均来源于历次 Roslyn CTP 中的精彩提问与解答。在许多情况下，社区成员无需团队介入便互相帮助，干得漂亮！你可以通过在本页搜索关键词或短语找到许多有用的内容。

在有代码可参考的情况下，问题的答案会包含一个或多个类似"FAQ(27)"的标签。你可以打开相应的代码文件并搜索该标签。由于没有在本文档中列出所有代码，文档不太可能与实际可运行的代码产生偏差。示例/测试项目始终与任何 API 变更保持同步。


## 目录

* [项目 / 跨领域问题](#project-/-cross-cutting-questions)
    * [Roslyn 有哪些可用文档](#what-docs-are-available-on-roslyn)
    * [我可以在编译器管道中重写源代码吗](#can-i-rewrite-source-code-within-the-compiler-pipeline)
    * [我可以在博客或示例代码中重新分发 Roslyn DLL 吗](#can-i-redistribute-the-roslyn-dlls-with-my-samples-or-code-on-my-blog)
    * [Roslyn API 与 VS 代码模型和 CodeDom 的关系是什么](#how-do-the-roslyn-apis-relate-to-the-vs-code-model-and-codedom)
    * [你能帮我提交一个 Connect bug 吗](#can-you-just-open-a-connect-bug-for-me)
* [GitHub 站点](#github-site)
    * [为什么有几个解决方案文件？](#why-are-there-several-solution-files)
    * [我可以在 Visual Studio 中本地运行哪些组件？](#what-components-can-i-run-locally-in-visual-studio)
* [获取信息的问题](#getting-information-questions)
    * [如何在声明中获取变量的类型信息（推断的 'var' 或显式变量类型）](#how-do-i-get-type-info-for-a-variable-in-a-declaration-with-inferred-var-or-explicit-variable-type)
    * [如何获取在给定代码位置可用的所有指定类型的已声明变量](#how-do-i-get-all-variables-declared-of-a-specified-type-that-are-available-at-a-given-code-locations)
    * [如何获取代码位置的补全列表或可访问符号](#how-do-i-get-a-completion-list-or-accessible-symbols-at-a-code-location)
    * [如何获取包含可访问类型成员的补全列表](#how-do-i-get-a-completion-list-with-members-of-an-accessible-type)
    * [如何获取调用方/被调用方信息](#how-do-i-get-caller/callee-info)
    * [如何从解决方案出发查找符号/类型的所有引用](#how-do-i-go-from-a-solution-to-find-all-references-on-a-symbol/type)
    * [如何查找编译中对特定命名空间的所有调用](#how-do-i-find-all-calls-in-a-compilation-into-a-particular-namespace)
    * [如何获取程序集（或所有引用程序集）的所有符号](#how-do-i-get-all-symbols-of-an-assembly-or-all-referenced-assemblies)
    * [如何获取表达式节点的类型](#how-do-i-get-the-type-of-an-expression-node)
    * [如何通过公共 API 获取参数和已声明局部变量的类型信息](#how-do-i-get-type-information-of-parameters-and-declared-locals-with-common-api)
    * [如何从语义模型中获取标识符（或 IdentifierNameSyntax 节点）的类型信息（TypeSymbol）](#how-do-i-get-the-type-information-typesymbol-from-a-semantic-model-for-an-identifier-or-an-identifiernamesyntax-node)
    * [如何比较语法节点（可选择忽略附加的 trivia）](#how-do-i-compare-syntax-nodes-optionally-ignoring-attached-trivia)
    * [注释如何存储在语法树中（以及如何使用语法可视化工具）](#how-are-comments-stored-in-the-syntax-tree-and-how-to-use-the-syntax-visualizer)
    * [什么是结构化 trivia，如何访问它](#what-is-structured-trivia-and-how-do-i-get-at-it)
    * [是否有语法树可视化工具或可视化检查树的工具](#is-there-a-syntax-tree-visualization-or-tools-to-visually-inspect-a-tree)
    * [如何判断与符号关联的类型是否为已知类型？我必须自己构造 AssemblyQualifiedName 吗](#how-do-i-tell-if-the-type-associated-with-a-symbol-is-a-known-type--do-i-have-to-construct-the-assemblyqualifiedname-myself)
    * [如何判断符号是否相同](#how-do-i-tell-if-symbols-are-the-same)
    * [如何测试语义模型是否可以为语法节点提供信息](#how-can-i-test-if-a-semantic-model-can-provide-information-about-a-syntax-node)
    * [如何获取 ISymbol 的元数据 token](#how-can-i-get-the-metadata-token-for-an-isymbol)
    * [标识符（SyntaxTokens）为什么使用 Value 而非 Name，Value、ValueText 和 GetText 之间有什么关系](#what's-with-identifier's-syntaxtokens-having-value-rather-than-name,-and-how-do-value,-valuetext,-and-gettext-relate)
    * [为什么使用 ChildNodesAndTokens 而不是直接使用 Children](#why-use-childnodesandtokens-rather-than-just-children)
    * [如何获取行列信息以报告错误](#how-do-i-get-line-and-column-information-to-report-errors)
    * [如何查找特定类型的所有语法子树](#how-do-i-find-all-syntactic-sub-trees-of-a-particular-kind)
    * [如何获取定义的完全限定类型名称](#how-do-i-get-the-fully-qualified-type-name-of-a-definition)
    * [如何确定在给定调用点绑定的是哪个重载](#how-do-i-determine-which-overload-binds-at-a-given-call-site)
    * [如何判断表达式的 TypeSymbol 是否可以赋值给某个位置的 TypeSymbol](#how-can-i-tell-if-a-typesymbol-from-an-expression-can-be-assigned-to-the-typesymbol-of-a-location)
    * [如何获取局部变量声明的完全限定类型名称，而不仅仅是 SyntaxTree 中解析到的文本](#how-can-i-get-a-fully-qualified-type-name-for-a-local-variable-declaration,-instead-of-just-the-text-parsed-in-the-syntaxtree)
    * [如何获取 .NET Framework 版本](#how-do-i-get-the-.net-framework-version)
    * [如何获取项目的程序集符号、引用以及项目中每个文档或项的语法树](#how-do-i-get-a-project's-assembly-symbol,-references,-and-syntax-trees-for-each-document-or-item-in-the-project)
    * [如何提取特定（子）类型的注解](#how-do-i-extract-an-annotation-of-a-particular-sub-type)
    * [为什么 Workspace API 不支持更多 VS 概念（嵌套文档或非代码文件）](#why-doesnt-the-workspace-api-support-more-vs-concepts-nested-documents-or-non-code-files)
    * [如何获取基类型或已实现接口信息，以及哪些成员重写或实现了基成员](#how-do-i-get-base-type-or-implemented-interface-information-and-which-members-override-or-implement-base-members)
    * [如何使用符号查找和调查已应用于方法的特性](#how-do-i-use-symbols-to-find-and-investigate-attributes-that-have-been-applied-to-methods)
* [构建和更新树的问题](#constructing-and-updating-tree-questions)
    * [如何向类中添加方法](#how-do-i-add-a-method-to-a-class)
    * [如何替换子表达式、声明等](#how-do-i-replace-a-sub-expression,-declaration,-etc.)
    * [如何在声明处和所有引用处更改符号的名称](#how-do-i-change-the-name-of-a-symbol-at-the-declaration-site-and-all-reference-sites)
    * [我可以向语法和符号添加自定义信息吗](#can-i-add-custom-information-to-syntax-and-symbols)
    * [如何使用 SyntaxRewriter 删除语句](#how-can-i-remove-a-statement-with-a-syntaxrewriter)
    * [如何根据另一个类型构造指针类型或数组类型](#how-do-i-construct-a-pointer-type-or-array-type-given-another-type)
    * [如何使用 SyntaxRewriter 删除 #region 和 #endregion（结构化 trivia）](#how-can-i-remove-#region-and-#endregion-structured-trivia-with-syntaxrewriter)
    * [如何向特定类型的所有语句添加日志记录（例如，记录变量内容）](#how-can-i-add-logging-to-all-statements-of-a-particular-kind-for-example,-to-log-contents-of-variables)
    * [如何删除代码文件中的所有注释](#how-can-i-remove-all-comments-from-a-file-of-code)
* [脚本、REPL 和执行代码的问题](#scripting,-repl,-and-executing-code-questions)
    * [REPL 和宿主脚本 API 怎么了](#what-happened-to-the-repl-and-hosting-scripting-apis)
    * [Roslyn API 与 LINQ 表达式树或表达式树 v2 的关系是什么？哪个更适合元编程或实现 DSL](#how-do-the-roslyn-apis-relate-to-linq-expression-trees-or-expression-trees-v2--is-one-better-for-meta-programming-or-implementing-dsls)
    * [如何将代码编译为可回收类型或 DynamicMethod](#how-do-i-compile-some-code-into-a-collectible-type-or-dynamicmethod)
* [其他问题](#miscellaneous-questions)
    * [什么是弹性 trivia](#what-is-elastic-trivia)
    * [为什么某些 Syntax 工厂方法会附加我未请求的弹性 trivia](#why-do-some-syntax-factories-attach-elastic-trivia-i-didn't-ask-for)
    * [如何将树或节点格式化为文本表示](#how-do-i-format-a-tree-or-node-to-a-textual-representation)
    * [如何在 MSBuild 任务中使用 Roslyn 并避免元数据获取和重入冲突](#how-can-i-use-roslyn-in-an-msbuild-task-and-avoid-metadata-fetching-and-re-entrancy-conflicts)
    * [是否有将程序编译为 IL 的端到端示例（Emit API）](#is-there-an-end-to-end-example-on-compiling-a-program-to-il-emit-apis)
    * [如何从编译中捕获 IL、调试信息和文档注释输出](#how-can-i-capture-il,-debug-info,-and-doc-comment-outputs-from-a-compilation)
    * [是否有 Roslyn 类型的对象模型图或类型继承关系图](#is-there-an-object-model-chart-or-type-inheritance-diagram-of-roslyn-types)
    * [如何在 Windows 8 上构建](#how-to-build-on-windows-8)

## 项目 / 跨领域问题

## Roslyn 有哪些可用文档？
有一些特性规范、语言特性讨论的设计说明以及文档注释的完整覆盖。但是，团队目前没有最新的 API 文档。请参阅 [文档]。

### 我可以在编译器管道中重写源代码吗？
Roslyn 不提供贯穿整个编译器管道的插件架构，无法在每个阶段影响语法解析、语义分析、优化算法、代码生成等。但是，你可以使用预构建规则来分析并生成不同的代码，MSBuild 随后将其传递给 csc.exe 或 vbc.exe。你可以使用 Roslyn 解析代码并进行语义分析，然后重写语法树、更改引用等，再将结果作为新的编译进行编译。

### 我可以重新分发 Roslyn DLL 吗？
可以。重新分发 Roslyn DLL 的推荐方式是使用 [Roslyn NuGet 包](http://www.nuget.org/packages/Microsoft.CodeAnalysis)。

### Roslyn API 与 VS 代码模型和 CodeDom 的关系是什么？
CodeDom 面向服务器端 ASP.NET 中的程序化代码生成和编译场景。后来被一些工具用途所借用，在代码生成功能基础上增加了对现有代码的建模。

VS 代码模型旨在帮助 ISV 避免在 VS 中解析代码，使 ISV 能够向 VS 提供面向代码的扩展。VS 代码模型将代码（并提供有限的代码生成支持）表示到类型、成员和参数的层级。它不深入到函数内部的语句层级。

Roslyn API 完整地建模了所有 C# 和 VB 代码，并提供完整的代码生成或更新能力。在 Visual Studio 2015 中，VS 代码模型 API 是构建在 Roslyn API 之上的。CodeDom 是由 C# 和 VB 团队之外的人员构建的独立技术，无需重写（尽管有人可能想将其迁移到新的 Roslyn API）。

### 你能帮我提交一个 Connect bug 吗？
作为微软员工，在论坛或通过邮件与大家交流时，我们很乐意帮你提交内部 bug 以减少你的工作量。如果我们说会提交 bug，那么你可以确信我们已经提交，并会给予该问题与任何客户问题同等的重视。如果你想跟踪该 bug 并自动收到任何变更的通知，则需要你自己提交 Connect bug。

我们请求你自己提交 Connect bug 而不是由我们代劳，有几个关键原因。首先，我们尽量避免人为地推高我们自己功能的客户需求量，因此有时在审查 Connect bug 时，审计员会询问某个 bug 是否真的来自客户。其次，Connect bug 为报告问题的客户保留了一个沟通渠道，以备我们需要进一步了解重现信息或场景说明时使用。

当员工代你提交 Connect issue 时，任何更新都会导致 Connect 向该员工发送邮件，而不是向客户发送，一旦该 bug 转由提交 bug 的员工以外的其他团队处理，这就会成为问题。客户信息或联系方式可能会丢失。尽管如此，这并非强制性政策，在我们与客户有长期合作伙伴关系的特定情况下，我们会代表客户提交 Connect bug。这不是我们的常规工作流程。

我们确实希望让向我们提供反馈的客户能够非常便捷地记录反馈，所以我们倾向于主动帮你提交。但如果你想收到更新邮件并跟踪该 bug，我们请求你自行提交 Connect bug。有一个 [url:VS Feedback|http://visualstudiogallery.msdn.microsoft.com/f8a5aac8-0418-4f88-9d34-bdbe2c4cfe72] 工具，专门设计用于更方便地报告 Connect 问题。

## GitHub 站点
### 为什么有几个解决方案文件？
主解决方案 Roslyn.slnx 包含由编译器、工作区和 Visual Studio 层组成的完整代码库。构建 Roslyn.slnx 需要 Visual Studio 和兼容版本的 VS SDK。Compilers.slnf 解决方案筛选器仅包含编译器层，因此可以在没有 Visual Studio 或 VS SDK 的情况下构建。

### 我可以在 Visual Studio 中本地运行哪些组件？
从 Visual Studio 2015 Update 1 开始，Roslyn 的所有部分都可以在 Visual Studio 中运行。请阅读我们的 [Windows 上的构建](https://github.com/dotnet/roslyn/blob/main/docs/contributing/Building%2C%20Debugging%2C%20and%20Testing%20on%20Windows.md)说明获取更多信息。

## 获取信息的问题

### 如何在声明中获取变量的类型信息（推断的 'var' 或显式变量类型）？
请参阅标记为"FAQ(1)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何从将 'var' 建模为标识符的 IdentifierNameSyntax（C#）或类型名称标识符（VB）中获取类型。请参阅标记为"FAQ(2)"的示例代码答案，了解如何从对整个 'var' 声明（C#）或带有 Option Infer on 的 'dim' 声明（VB）建模的 VariableDeclaratorSyntax 中获取类型。

### 如何获取在给定代码位置可用的所有指定类型的已声明变量？
请参阅标记为"FAQ(4)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何获取指定类型的名称，包括如何筛选不同类型的成员或作用域。SemanticModel.LookupSymbols API 将提供在给定位置可通过简单名称引用的所有符号。有几个注意事项。

LookupSymbols 可能返回在 C# 中具有不可言说名称的符号，例如匿名类型。你可以检查符号属性 CanBeReferencedByName。

结果包括局部符号和更广范围的符号，例如方法、字段、类型和命名空间的符号。如果你只关心局部变量，可以检查返回符号的 Kind，参见代码示例。

LookupSymbols 可能返回无法引用但技术上在作用域内的符号。例如，由于 C# 将词法提升到函数作用域，你可能会找到在函数后面声明的局部变量的符号，但在声明之前引用它在语义上是非法的。

### 如何获取代码位置的补全列表或可访问符号？
请参阅标记为"FAQ(4)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何获取位置处的可访问名称，包括如何筛选不同类型的成员或作用域。此问题类似于"如何获取在给定代码位置可用的所有指定类型的已声明变量？"，但请注意，真正获取 Visual Studio 在补全列表中显示的内容有多个需要处理的问题。

### 如何获取包含可访问类型成员的补全列表？
请参阅标记为"FAQ(5)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何获取可访问成员。

### 如何获取调用方/被调用方信息？
请参阅标记为"FAQ(6)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何获取调用方/被调用方信息。注意，该示例不一定完整；例如，被分析的代码可能已将函数赋值给委托变量然后调用它，示例中未对此进行处理。

### 如何从解决方案出发查找符号/类型的所有引用？
请参阅标记为"FAQ(7)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何获取引用。简而言之，你应该使用 Microsoft.CodeAnalysis.SymbolFinder API。

### 如何查找编译中对特定命名空间的所有调用？
请参阅标记为"FAQ(8)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何查找对特定命名空间的所有调用（或来自该命名空间的函数调用）。

### 如何获取程序集（或所有引用程序集）的所有符号？
请参阅标记为"FAQ(9)"和"FAQ(10)"的示例代码答案（[安装位置信息|faq#codefiles]）。前者展示了遍历所有命名空间符号、类型符号，以及为简单起见遍历字段和方法定义。后者展示了一个用于遍历所有表达式以发现引用符号的 walker。

### 如何获取表达式节点的类型？
请参阅标记为"FAQ(3)"的示例代码答案（[安装位置信息|faq#codefiles]），了解获取表达式类型信息的几个示例。

### 如何通过公共 API 获取参数和已声明局部变量的类型信息？
请参阅标记为"FAQ(3)"的示例代码答案（[安装位置信息|faq#codefiles]），了解获取变量类型信息的几个示例。你会注意到 Roslyn 对参数和其他局部变量的建模方式不同，分别使用 LocalSymbol 和 ParameterSymbol，这是合理的，因为参数可以是 ref/out、有默认值等。

### 如何从语义模型中获取标识符（或 IdentifierNameSyntax 节点）的类型信息（TypeSymbol）？
请参阅标记为"FAQ(1)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何从将 'var' 建模为标识符的 IdentifierNameSyntax 中获取类型。

### 如何比较语法节点（可选择忽略附加的 trivia）？
请参阅标记为"FAQ(11)"的示例代码答案（[安装位置信息|faq#codefiles]），了解比较树和节点的几个方面。

### 注释如何存储在语法树中（以及如何使用语法可视化工具）？
Roslyn 树将注释作为 trivia 存储在 token 上。所使用的启发式规则是将 trivia 附加到其后面的 token（即作为包含该 token 的节点的前置 trivia）。但是，如果 trivia 位于某行的最后一个 token 之后，则附加到前面的 token（即作为尾随 trivia）。

在以下示例中，第一个注释作为 LeadingTrivia 附加到其后面声明中的 'int' token。你也可以在包含 'int' token 的 LocalDeclarationStatementSyntax 节点上看到前置 trivia。第一个注释 trivia 附加到其后面的第一个 token，尽管 'int' 在不同的行上，因为注释所在的行上没有 token。

``` csharp
int method1 () {
  int a = 1;
  // comment1
  int b = 1; // comment 2
}
```

第二个注释作为 TrailingTrivia 附加到分号上，你也可以在 LocalDeclarationStatementSyntax 节点上看到它。

有关允许你直观检查代码片段并了解 Roslyn 如何构建语法树的语法可视化工具的概述，请参见[此处|Syntax Visualizer]。

请参阅标记为"FAQ(29)"的示例代码答案（[安装位置信息|faq#codefiles]），了解一个收集代码中注释并区分结构化文档注释的小型 SyntaxWalker。

### 什么是结构化 trivia，如何访问它？
大多数 trivia（如 WhitespaceTrivia 或 SingleLineCommentTrivia）仅通过 Kind 属性区分 trivia 的类型。所有 trivia 都具有相同的结构类型 SyntaxTrivia。

某些 trivia（如指令或 xml 文档注释）具有带有 SyntaxNode 派生类型的结构化模型来表示它们。你可以通过检查 HasStructure 属性来判断 trivia 是否具有结构。如果 trivia 有结构，可以对其调用 GetStructure 方法以获取 SyntaxNode。例如，如果你有一个 IfDirective trivia，调用其 GetStructure 方法将返回一个 IfDirectiveSyntax 节点。该节点允许你访问 HashToken、IfKeyword、条件的 ExpressionSyntax 节点等。

### 是否有语法树可视化工具或可视化检查树的工具？
有！请查看[此处|Syntax Visualizer]了解有关语法可视化工具的详细信息。

### 如何判断与符号关联的类型是否为已知类型？我必须自己构造 AssemblyQualifiedName 吗？
最好使用 Equals()。符号重载了 operator==，在许多情况下有效，但对公共符号 API（ISymbol）无法正常工作。将已知类型与符号类型进行比较的最简单方法是使用 Compilation.GetTypeByMetadataName 获取已知类型，然后用 Equals() 进行比较。请参阅标记为"FAQ(12)"的示例代码答案（[安装位置信息|faq#codefiles]），了解符号比较。

对于少量非常常见的类型（例如 Int32 和 String）以及编译器感兴趣的几种特殊类型，属性 TypeSymbol.SpecialType 可让你轻松判断你的已知类型是否是这些特殊类型之一。请参阅标记为"FAQ(1)"和"FAQ(3)"的示例代码答案，了解与特殊类型的符号比较。

比较来自不同编译的符号结果是未定义的。

### 如何判断符号是否相同？
最好使用 Equals()。符号重载了 operator==，在许多情况下有效，但对公共符号 API（ISymbol）无法正常工作。请参阅标记为"FAQ(12)"的示例代码答案（[安装位置信息|faq#codefiles]），了解符号比较。
即使使用完全相同的参数对象调用 API，也不能保证返回的是同一个 Symbol 对象。

对于少量非常常见的类型（例如 Int32 和 String）以及编译器感兴趣的几种特殊类型，属性 TypeSymbol.SpecialType 可让你轻松判断类型符号是否是这些特殊类型之一。请参阅标记为"FAQ(1)"和"FAQ(3)"的示例代码答案，了解与特殊类型的符号比较。

比较来自不同编译的符号结果是未定义的。

### 如何测试语义模型是否可以为语法节点提供信息？
请参阅标记为"FAQ(13)"的示例代码答案（[安装位置信息|faq#codefiles]），了解检查树、节点和语义模型是否相关的几种方法。

### 如何获取 ISymbol 的元数据 token？
这对于在 Roslyn API 和其他工具（如 ildasm）之间互操作可能很有用。目前我们没有办法做到这一点。

### 标识符（SyntaxTokens）为什么使用 Value 而非 Name，Value、ValueText 和 GetText 之间有什么关系？
当我考虑"ParameterList.Parameters{"[0]"}.Identifier.Value"时，Name 不是比 Value 更好的选择吗？

关于标识符使用 .Value 还是 .Name，我们使用 SyntaxToken 来表示标识符。SyntaxToken 也表示源代码中所有有意义的文本片段，如字面量、标点符号等。SyntaxTokens 没有名称的概念，但确实对文本进行建模。SyntaxToken.GetText() 返回解析源代码时读取的原始文本。SyntaxToken.Value 返回 SyntaxToken 所表示文本的对象"值"。SyntaxToken.ValueText 返回该值的文本表示。

考虑 "long @long = 1L"。转义标识符 "@long" 的 SyntaxToken.Value 和 SyntaxToken.ValueText 都返回字符串 "long"。但是，SyntaxToken.GetText() 返回字符串 "@long"。类似地，字面量 "1L" 的 SyntaxToken.Value 返回装箱的整数 "1"。SyntaxToken.ValueText 返回字符串 "1"。SyntaxToken.GetText() 返回从代码中解析的文本 "1L"。

请参阅标记为"FAQ(14)"的示例代码答案（[安装位置信息|faq#codefiles]）。

### 为什么使用 ChildNodesAndTokens 而不是直接使用 Children？
最初有一个 Children 属性。可用性测试的反馈表明，人们使用语法 API 的一个关键问题就是使用这个集合。这些名称已更改，以强调节点和 token 之间的区别，并引导人们避免默认使用 SyntaxNodeOrToken 类型。

### 如何获取行列信息以报告错误？
请参阅标记为"FAQ(16)"和"FAQ(17)"的示例代码答案（[安装位置信息|faq#codefiles]），了解从树和节点获取位置信息的几个示例。

### 如何查找特定类型的所有语法子树？
请参阅标记为"FAQ(18)"的示例代码答案（[安装位置信息|faq#codefiles]），了解一个访问特定类型节点和 token 的 SyntaxWalker。其他几个示例使用 node.DescendentNodes().OfType<>() 获取子树（例如"FAQ(1)"）。有些示例展示了使用 node.DescendentNodes().First(t => t.Kind == SyntaxKind…) 进行过滤，如"FAQ(11)"。

### 如何获取定义的完全限定类型名称？
请参阅标记为"FAQ(19)"的示例代码答案（[安装位置信息|faq#codefiles]），了解获取和比较名称的几个示例，包括开放泛型类型与封闭泛型类型的比较。

### 如何确定在给定调用点绑定的是哪个重载？
请参阅标记为"FAQ(20)"的示例代码答案（[安装位置信息|faq#codefiles]），了解解析重载的方法。

### 如何判断表达式的 TypeSymbol 是否可以赋值给某个位置的 TypeSymbol？
请参阅标记为"FAQ(21)"和"FAQ(22)"的示例代码答案（[安装位置信息|faq#codefiles]），了解确定可赋值性以及需要何种转换的几个示例。

### 如何获取局部变量声明的完全限定类型名称，而不仅仅是 SyntaxTree 中解析到的文本？
请参阅标记为"FAQ(19)"的示例代码答案（[安装位置信息|faq#codefiles]）。

### 如何获取 .NET Framework 版本？
请参阅标记为"FAQ(23)"的示例代码答案（[安装位置信息|faq#codefiles]），本质上是获取 SpecialType.System_Object 的包含程序集，然后获取程序集的版本。

### 如何获取项目的程序集符号、引用以及项目中每个文档或项的语法树？
请参阅标记为"FAQ(24)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何处理一个引用和文档，但你可以轻松地将代码转换为循环处理所有引用和文档。

### 如何提取特定（子）类型的注解？
请参阅标记为"FAQ(25)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何注解某些 token，然后查找具有特定类型注解的 token。该示例未展示，但这同样适用于节点。

### 为什么 Workspace API 不支持更多 VS 概念（嵌套文档或非代码文件）？
Workspace API 表示 C# 和 VB 语言服务实际使用的解决方案、项目和文档视图。Workspace 模型包括 C# 和 VB 源文件、引用、编译选项及其他类似详情。但是，Workspace API 不表示 Visual Studio 项目系统的每一个功能，例如 xaml、资源文件等的建模。Workspace 模型也可以独立于 Visual Studio 工作。

要获取 Visual Studio 拥有的所有项目项信息，请使用 .NET Framework 中提供的 MSBuild API。如果你的代码在 Visual Studio 内执行，也可以使用 IVsHierarchy API 枚举项目项，这将提供对 Visual Studio 管理的嵌套文档的访问。如果你想对 C# 和 VB 文件进行语义分析，应使用工作区。

### 如何获取基类型或已实现接口信息，以及哪些成员重写或实现了基成员？
请参阅标记为"FAQ(37)""FAQ(38)"的示例代码答案（[安装位置信息|faq#codefiles]），了解按名称获取类型、获取基类型和已实现类型，以及检查成员以获取虚拟、重写、密封等信息。

### 如何使用符号查找和调查已应用于方法的特性？
请参阅标记为"FAQ(39)"的示例代码答案（[安装位置信息|faq#codefiles]），了解处理特性的一些基本机制。

## 构建和更新树的问题

### 如何向类中添加方法？
请参阅标记为"FAQ(26)"的示例代码答案（[安装位置信息|faq#codefiles]）。

### 如何替换子表达式、声明等？
请参阅标记为"FAQ(26)"的示例代码答案（[安装位置信息|faq#codefiles]），了解替换类声明；以及"FAQ(27)"，了解替换子表达式。

注意，如果加载了 Roslyn 语言服务，Visual Studio 中可能同时有多个工作区处于活动状态：

* 如果加载了 Roslyn 语言服务，则有一个工作区对 Visual Studio 内的活动解决方案进行建模。
* 如果加载了 Roslyn 语言服务，则有一个工作区对在 VS 中打开的所有杂项（参见杂项文件项目）.cs 和 .vb 文件的解决方案和项目进行建模。
* 有一个工作区对在 VS 中打开的所有杂项 .csx 文件的解决方案和项目进行建模。
* 有一个工作区对交互式窗口中的提交链进行建模。

由于 Roslyn 语言服务功能（补全、代码问题、重构支持等）在工作区上运行，这些功能在上述所有工作区上下文中均可使用。

### 如何在声明处和所有引用处更改符号的名称？
请参阅标记为"FAQ(28)"的示例代码答案（[安装位置信息|faq#codefiles]），了解重命名示例。注意，该示例不处理泛型名称，根据你重命名的内容类型，你可能会以不同方式创建 SyntaxRewriter（例如，不访问类声明或构造函数声明）。

### 我可以向语法和符号添加自定义信息吗？
请参阅标记为"FAQ(25)"的示例代码答案（[安装位置信息|faq#codefiles]），了解如何注解某些节点，然后查找具有特定类型注解的节点。

对于符号没有此机制。

### 如何使用 SyntaxRewriter 删除语句？
请参阅标记为"FAQ(30)"的示例代码答案（[安装位置信息|faq#codefiles]），了解删除语句的方法，以及何时可以简单地从 visit 方法返回 null，何时需要返回 Syntax.EmptyStatement()。

### 如何根据另一个类型构造指针类型或数组类型？
请参阅标记为"FAQ(31)"的示例代码答案（[安装位置信息|faq#codefiles]）。
w

### 如何使用 SyntaxRewriter 删除 #region 和 #endregion（结构化 trivia）？
请参阅标记为"FAQ(32)"和"FAQ(33)"的示例代码答案（[安装位置信息|faq#codefiles]），了解删除这些指令的几种方法。

### 如何向特定类型的所有语句添加日志记录（例如，记录变量内容）？
请参阅标记为"FAQ(34)"的示例代码答案（[安装位置信息|faq#codefiles]）。

### 如何删除代码文件中的所有注释？
请参阅标记为"FAQ(32)"和"FAQ(33)"的示例代码答案（[安装位置信息|faq#codefiles]），了解删除 #region 指令的几种方法，但你可以将代码修改为查找 SyntaxKind SingleLineComment 或 MutliLineComment。

## 脚本、REPL 和执行代码的问题

### REPL 和宿主脚本 API 怎么了？
C# 交互式窗口已在 Visual Studio Update 1 中回归。尽情享用！

### Roslyn API 与 LINQ 表达式树或表达式树 v2 的关系是什么？哪个更适合元编程或实现 DSL？
表达式树 v2 是一个语义模型，其某些形状类似于语法。它们确实支持一些控制流、赋值语句、递归等。但是，它们不对 C# 和 VB 语言所具有的许多内容进行建模，例如类型定义。Roslyn API 将对 C# 和 VB 的语法、语义绑定、代码生成等具有完整的保真度。

虽然 DLR 和 ET v2 可能适合某些元编程，但它们的真正重点是支持 .NET 上的动态语言，以及使应用程序能够承载这些语言以用于应用程序脚本编写。

如果你的目标是嵌入式领域特定语言（DSL），Roslyn API 将更为适合，可能也是你唯一的考虑。嵌入式 DSL 是语言的语法扩展，你可以在编译期间或执行前提供核心语言代码，以赋予新语法语义。有关语言中 DSL 支持的示例，请参阅 [url:Dylan 语言|http://opendylan.org/books/dpg/db_329.html]。否则，DSL 只是高度特定于某个领域的正式语言，例如 SQL，而不是像 C# 这样的通用语言。Roslyn API 无疑限制更少，因为它们将支持 C# 和 VB 等语言所能表达的一切。

## 其他问题

### 什么是弹性 trivia？
弹性 trivia 通常出现在手动构建的语法树中，用于表示灵活的空白元素。解析器返回的树将任何空白字面量地表示为源代码中的原始内容。弹性空白允许生成的树建议空白元素，并确保 token 彼此不直接相邻。格式化工具和其他树处理工具可以自由地替换、延长或以任何方式更改弹性空白，而不会破坏与原始代码源的保真度。

### 为什么某些 Syntax 工厂方法会附加我未请求的弹性 trivia？
弹性空白表示建议的空白，但将树渲染为文本时，弹性空白处可能使用零个或多个空白。某些工厂方法添加弹性空白，以帮助避免意外地将两个内容拼接在一起（此处可能需要空白），或者工具可能希望应用格式化选项。当然，渲染器需要智能地在某些 token 之间放置一些空白，但它也可以根据格式化选项在开括号周围消除所有空白。

这种弹性 trivia 的报告长度为零，因为在将树映射到文本时，它可能会被省略。

如果需要，你可以对工厂方法的结果进行操作，删除弹性 trivia（尽管这种情况极为罕见），例如：

``` csharp
     Syntax.Token(SyntaxKind.SemicolonToken).WithLeadingTrivia().WithTrailingTrivia()
```

### 如何将树或节点格式化为文本表示？
请参阅标记为"FAQ(35)"的示例代码答案（[安装位置信息|faq#codefiles]），了解几个格式化示例。该示例还包括展示 Document 上的一些服务，例如 SimplifyNames()。示例"FAQ(26)"也展示了 SyntaxNode.Format()。

### 如何在 MSBuild 任务中使用 Roslyn 并避免元数据获取和重入冲突？
目前对此没有很好的答案。Roslyn 在内部使用 MSBuild 构建工作区，所以当你试图在任务中构建它们时，会出现重入冲突问题。即使它真的可以工作，也会存在大量重复工作和空间浪费的问题。正确的方式是创建一个基础 RoslynTask，继承自 Task，并使用已经存在的 MSBuild 信息。作为临时解决方案，你可以从 MSBuild Task 中的一些信息创建 Roslyn 编译，然后使用编译的代码信息。但这仍然不是完整的解决方案，因为你可能想影响正在编译的代码、向 MSBuild 目标添加文件或删除文件。

以下代码不在示例/测试项目中，因为它不是一个很好的答案，但由于这个问题出现得足够频繁，可以给你提供一些参考：

``` csharp
public class MyTask : Task {
    public override bool Execute() {
          var projectFileName = this.BuildEngine.ProjectFileOfTaskNode;
              var project = ProjectCollection.GlobalProjectCollection.
                            GetLoadedProjects(projectFileName).Single();
              var compilation = CSharpCompilation.Create(
                                    project.GetPropertyValue("AssemblyName"),
                                    syntaxTrees: project.GetItems("Compile").Select(
                                      c => SyntaxFactory.ParseCompilationUnit(
                                               c.EvaluatedInclude).SyntaxTree),
                                    references: project.GetItems("Reference")
                                                       .Select(          
                                      r => new MetadataFileReference
                                                   (r.EvaluatedInclude)));
             // Now work with compilation ...
    }
}
```

### 是否有将程序编译为 IL 的端到端示例（Emit API）？
请参阅下一个问题，[#如何从编译中捕获 IL、调试信息和文档注释输出]。

### 如何从编译中捕获 IL、调试信息和文档注释输出？
请参阅标记为"FAQ(34)"的示例代码答案（[安装位置信息|faq#codefiles]）。该示例包含一个 Execute() 方法定义，用于编译和执行二进制文件。

### 是否有 Roslyn 类型的对象模型图或类型继承关系图？
你可以创建一个可缩放和搜索的类型继承关系图。你需要 Visual Studio 2010 Ultimate，创建关系图的说明在这篇[帖子](http://social.msdn.microsoft.com/Forums/en-US/roslyn/thread/705b090b-58ac-4a94-b7b5-d1408205bc90)中。

### 如何在 Windows 8 上构建

如果你没有安装 Visual Studio，可能需要安装 .NET Framework（例如 4.6.2）。

然后在构建时可能会遇到错误 `CSC error CS0041: Unexpected error writing debug information -- 'DLL 'Microsoft.DiaSymReader.Native.amd64.dll' failed: the specified module could not be found. (Exception from HRESULT is returned: 0x8007007E)`。可以通过安装 [C 运行时](https://support.microsoft.com/en-us/help/2999226/update-for-universal-c-runtime-in-windows)（通用 CRT）来解决此问题。
