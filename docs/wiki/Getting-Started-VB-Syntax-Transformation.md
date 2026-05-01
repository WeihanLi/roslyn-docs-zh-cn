## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)
* [VB 语法分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-VB-Syntax-Analysis.md)
* [VB 语义分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-VB-Semantic-Analysis.md)

## 简介
本演练基于**入门：语法分析**和**入门：语义分析**演练中探索的概念和技术。如果你尚未完成这些演练，强烈建议在开始本演练之前先完成它们。

在本演练中，你将探索创建和转换语法树的技术。结合你在之前**入门**演练中学到的技术，你将创建你的第一个命令行重构！

## 不可变性与 .NET Compiler Platform
.NET Compiler Platform 的一个基本原则是不可变性。因为不可变的数据结构在创建后不能被更改，所以它们可以被多个使用者同时安全地共享和分析，而不会有一个工具以不可预测的方式影响另一个工具的危险。无需锁或其他并发措施。这适用于语法树、编译、符号、语义模型以及你将遇到的每个其他数据结构。修改不是直接在原对象上进行的，而是根据对旧对象的指定差异创建新对象。你将把这个概念应用到语法树上来创建树转换！

## 创建和"修改"树
### 使用工厂方法创建节点
要创建 **SyntaxNodes**，必须使用 **SyntaxFactory** 类的工厂方法。对于每种节点、标记或 trivia，都有一个工厂方法可用于创建该类型的实例。通过以自底向上的方式按层次组合节点，你可以创建语法树。 

#### 示例 - 使用工厂方法创建 SyntaxNode
此示例使用 **SyntaxFactory** 类的工厂方法构建表示 **System.Collections.Generic** 命名空间的 **NameSyntax**。

**NameSyntax** 是 VB 中出现的四种名称类型的基类： 

* **IdentifierNameSyntax** 代表简单的单标识符名称，如 **System** 和 **Microsoft**
* **GenericNameSyntax** 代表泛型类型或方法名称，如 **List(Of Integer)**
* **QualifiedNameSyntax** 代表 ```<左名称>.<右标识符或泛型名称>``` 形式的限定名称，如 **System.IO**
* **GlobalNameSyntax** 代表 **Global** 命名空间的名称。
通过将这些名称组合在一起，你可以创建 VB 语言中可以出现的任何名称。 

1) 创建一个新的 Visual Basic **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual Basic -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**ConstructionVB**"并点击确定。 

3) 在 **Module1.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off
```

  * 某些读者可能在项目级别默认启用了 **Option Strict**。在本演练中将 **Option Strict** **关闭**可以通过减少大量类型转换需求来简化许多示例。

4) 将以下 Imports 语句添加到文件顶部，以导入 **SyntaxFactory** 类的工厂方法，这样我们以后就可以不加限定地使用它们：
```VB.NET
Imports Microsoft.CodeAnalysis.VisualBasic.SyntaxFactory
```

5) 将光标移至 **Main** 方法的 **End Sub** 所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

6) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

7) 在 Main 方法内，创建一个简单的 **IdentifierNameSyntax**，代表 **System** 命名空间的名称，并将其赋值给变量。当你从这个节点构建 **QualifiedNameSyntax** 时，你将重用此变量，所以将此变量声明为 **NameSyntax** 类型，以允许它存储两种类型的 **SyntaxNode**——**不要**使用类型推断：
```VB.NET
        Dim name As NameSyntax = IdentifierName("System")
```

8) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

9) 打开即时窗口。
  * 在 Visual Studio 中，选择 **Debug -> Windows -> Immediate**。

10) 使用即时窗口，输入表达式 **? name.ToString()** 并按 Enter 求值。你应该会看到字符串"**System**"作为结果。
  * 注意你也可以在即时窗口中输入要执行的语句。在 Visual Basic 即时窗口中，必须以问号 **?** 开头来区分表达式求值与语句执行。 

11) 接下来，使用此 **name** 节点作为名称的**左**侧，使用 **Collections** 命名空间的新 **IdentifierNameSyntax** 作为 **QualifiedNameSyntax** 的**右**侧，构造一个 **QualifiedNameSyntax**：
```VB.NET
        name = QualifiedName(name, IdentifierName("Collections"))
```

12) 执行此语句，将 **name** 变量设置为新的 **QualifiedNameSyntax** 节点。

13) 使用即时窗口求值表达式 **? name.ToString()**。它应该求值为"**System.Collections**"。

14) 通过为 **Generic** 命名空间构建另一个 **QualifiedNameSyntax** 节点来继续此模式：
```VB.NET
        name = QualifiedName(name, IdentifierName("Generic"))
```

15) 执行此语句，再次使用即时窗口观察 **? name.ToString()** 现在求值为完全限定名称"**System.Collections.Generic**"。 

### 使用 With* 和 ReplaceNode 方法修改节点
因为语法树是不可变的，**Syntax API** 没有提供在构造后直接修改现有语法树的机制。然而，**Syntax API** 确实提供了基于对现有树的指定更改来生成新树的方法。从 **SyntaxNode** 派生的每个具体类都定义了 **With*** 方法，你可以使用它们来指定对其子属性的更改。此外，**ReplaceNode** 扩展方法可用于替换子树中的后代节点。如果没有此方法，更新节点还需要手动更新其父节点以指向新创建的子节点，并在整棵树中重复此过程——这个过程称为_重新旋转_树。 

#### 示例 - 使用 With* 和 ReplaceNode 方法进行转换
此示例使用 **WithName** 方法将 **ImportsStatementSyntax** 节点中的名称替换为上面构造的名称。

1) 继续上面的示例，添加以下代码以解析示例代码文件：
```VB.NET
        Dim tree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections
Imports System.Linq
Imports System.Text
 
Namespace HelloWorld
    Module Module1
        Sub Main(args As String())
            Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")
 
        Dim root As CompilationUnitSyntax = tree.GetRoot()
```

  * 注意该文件使用 **System.Collections** 命名空间而不是 **System.Collections.Generic** 命名空间。 

2) 执行这些语句。

3) 使用 **SimpleImportsClauseSyntax.WithName** 方法创建一个新的 **SimpleImportsClauseNode** 节点，用上面创建的名称更新"**System.Collections**"导入：
```VB.NET
        Dim oldImportClause As SimpleImportsClauseSyntax = 
                root.Imports(1).ImportsClauses(0)
 
        Dim newImportClause = oldImportClause.WithName(name)
```

4) 执行这些语句。

5) 使用即时窗口求值表达式 **? root.ToString()**，并观察原始树未被更改以包含此新的更新节点。

6) 添加以下行，使用 **ReplaceNode** 扩展方法创建一个新树，将现有导入替换为更新后的 **newImportClause** 节点，并将新树存储在现有 **root** 变量中：
```VB.NET
        root = root.ReplaceNode(oldImportClause, newImportClause)
```

7) 执行此语句。

8) 使用即时窗口求值表达式 **? root.ToString()**，这次观察树现在正确导入了 **System.Collections.Generic** 命名空间。

9) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop Debugging**。

10) 你的 **Module1.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Imports Microsoft.CodeAnalysis.VisualBasic.SyntaxFactory


Module Module1

    Sub Main()
        Dim name As NameSyntax = IdentifierName("System")

        name = QualifiedName(name, IdentifierName("Collections"))

        name = QualifiedName(name, IdentifierName("Generic"))

        Dim tree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections
Imports System.Linq
Imports System.Text
 
Namespace HelloWorld
    Module Module1
        Sub Main(args As String())
            Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")

        Dim root As CompilationUnitSyntax = tree.GetRoot()

        Dim oldImportClause As SimpleImportsClauseSyntax =
                root.Imports(1).ImportsClauses(0)

        Dim newImportClause = oldImportClause.WithName(name)

        root = root.ReplaceNode(oldImportClause, newImportClause)
    End Sub

End Module
```

### 使用 SyntaxRewriter 转换树
**With*** 和 **ReplaceNode** 方法提供了转换语法树单个分支的便捷手段。然而，通常可能需要在语法树上协调执行多个转换。**SyntaxRewriter** 类是 **SyntaxVisitor** 的子类，可用于将转换应用于特定类型的 **SyntaxNode**。还可以将一组转换应用于语法树中任何位置出现的多种类型的 **SyntaxNode**。以下示例在一个简单的命令行重构实现中演示了这一点，该重构在可以使用类型推断的任何地方删除局部变量声明中的显式类型。此示例利用了本演练中讨论的技术以及**入门：语法分析**和**入门：语义分析**演练中的技术。 

#### 示例 - 创建 SyntaxRewriter 以转换语法树
1) 创建一个新的 Visual Basic **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual Basic -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**TransformationVB**"并点击确定。 

3) 在 **Module1.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off

Imports System.IO
```

4) 向项目添加新类文件。
  * 在 Visual Studio 中，选择 **Project -> Add Class...** 
  * 在"添加新项"对话框中输入 **TypeInferenceRewriter.vb** 作为文件名。

5) 在 **TypeInferenceRewriter.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off
```

6) 使 **TypeInferenceRewriter** 类继承 **VisualBasicSyntaxRewriter** 类：
```VB.NET
Public Class TypeInferenceRewriter
    Inherits VisualBasicSyntaxRewriter
```

7) 添加以下代码，声明一个私有只读字段来保存 **SemanticModel**，并从构造函数中初始化它。你之后将需要此字段来确定可以在哪里使用类型推断：
```VB.NET
    Private ReadOnly SemanticModel As SemanticModel
 
    Public Sub New(semanticModel As SemanticModel)
        Me.SemanticModel = semanticModel
    End Sub
```

8) 重写 **VisitLocalDeclarationStatement** 方法：
```VB.NET
    Public Overrides Function VisitLocalDeclarationStatement (
                                  node As LocalDeclarationStatementSyntax
                              ) As SyntaxNode

    End Function
```

  * 注意 **VisitLocalDeclarationStatement** 方法返回 **SyntaxNode**，而不是 **LocalDeclarationStatementSyntax**。在此示例中，你将基于现有节点返回另一个 **LocalDeclarationStatementSyntax** 节点。在其他场景中，一种节点可能完全被另一种节点替换——甚至被删除。

9) 出于本示例的目的，你只处理局部变量声明，尽管类型推断也可以用于 **For Each** 循环、**For** 循环、LINQ 表达式和 Lambda 表达式。此外，此重写器只转换最简单形式的声明：
```VB.NET
Dim variable As Type = expression
```

VB 中以下形式的变量声明要么与类型推断不兼容，要么留给读者作为练习。

```VB.NET
' Multiple types in a single declaration.
Dim variable1 As Type1 = expression1, 
    variable2 As Type2 = expression2
' Multiple variables in a single declaration.
Dim variable1, variable2 As Type 
' No initializer.
Dim variable1 As Type 
Dim variable As New Type
' Already inferred.
Dim variable = expression
```

10) 将以下代码添加到 **VisitLocalDeclarationStatement** 方法主体中，以跳过重写这些形式的声明：
```VB.NET
        If node.Declarators.Count > 1 Then Return node
        If node.Declarators(0).Names.Count > 1 Then Return node
        If node.Declarators(0).AsClause Is Nothing Then Return node
        If node.Declarators(0).AsClause.Kind = SyntaxKind.AsNewClause _
            Then Return node
        If node.Declarators(0).Initializer Is Nothing Then Return node
```

  * 注意返回未修改的 **node** 参数会导致该节点不发生重写。 

11) 添加这些语句以提取声明中指定的类型名称，并使用 **SemanticModel** 字段对其进行绑定以获取类型符号。
```VB.NET
        Dim declarator As VariableDeclaratorSyntax = node.Declarators(0)
        Dim asClause As SimpleAsClauseSyntax = declarator.AsClause
        Dim variableTypeName As TypeSyntax = asClause.Type
 
        Dim variableType As ITypeSymbol = 
                SemanticModel.GetSymbolInfo(variableTypeName).Symbol
```

12) 现在，添加此语句以绑定初始化器表达式：
```VB.NET
        Dim initializerInfo As TypeInfo = 
                SemanticModel.GetTypeInfo(declarator.Initializer.Value)
```

13) 最后，添加以下 If 语句，如果初始化器表达式的类型与 **As** 子句中指定的类型匹配，则删除 **As** 子句：
```VB.NET
        If variableType Is initializerInfo.Type Then
 
            Dim newDeclarator As VariableDeclaratorSyntax =
                    declarator.WithAsClause(Nothing)
 
            Return node.ReplaceNode(declarator, newDeclarator)
        Else
            Return node
        End If
```

  * 注意此条件是必需的，因为如果类型不匹配，声明可能正在将初始化器表达式转换为基类或接口，或执行隐式转换。在这些情况下删除显式类型将改变程序的语义。
  * 注意将 **Nothing** 传递给 **WithAsClause** 方法会导致现有 **As** 子句被删除。
  * 还要注意，使用 **ReplaceNode** 而不是 **With*** 方法来转换 **LocalDeclarationStatementSyntax** 更简单，因为 **LocalDeclarationStatementSyntax** 节点保存的是 **SeparatedSyntaxList(Of VariableDeclaratorSyntax)**。使用 **ReplaceNode** 避免了作为中间步骤构建和填充新列表的需求。 

14) 你的 **TypeInferenceRewriter.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Public Class TypeInferenceRewriter
    Inherits VisualBasicSyntaxRewriter

    Private ReadOnly SemanticModel As SemanticModel

    Public Sub New(semanticModel As SemanticModel)
        Me.SemanticModel = semanticModel
    End Sub

    Public Overrides Function VisitLocalDeclarationStatement(
                                  node As LocalDeclarationStatementSyntax
                              ) As SyntaxNode
        If node.Declarators.Count > 1 Then Return node
        If node.Declarators(0).Names.Count > 1 Then Return node
        If node.Declarators(0).AsClause Is Nothing Then Return node
        If node.Declarators(0).AsClause.Kind = SyntaxKind.AsNewClause _
            Then Return node
        If node.Declarators(0).Initializer Is Nothing Then Return node

        Dim declarator As VariableDeclaratorSyntax = node.Declarators(0)
        Dim asClause As SimpleAsClauseSyntax = declarator.AsClause
        Dim variableTypeName As TypeSyntax = asClause.Type

        Dim variableType As ITypeSymbol =
                SemanticModel.GetSymbolInfo(variableTypeName).Symbol

        Dim initializerInfo As TypeInfo =
                SemanticModel.GetTypeInfo(declarator.Initializer.Value)

        If variableType Is initializerInfo.Type Then

            Dim newDeclarator As VariableDeclaratorSyntax =
                    declarator.WithAsClause(Nothing)

            Return node.ReplaceNode(declarator, newDeclarator)
        Else
            Return node
        End If
    End Function
End Class
```

15) 返回 **Module1.vb** 文件。

16) 要测试你的 **TypeInferenceRewriter**，你需要创建一个测试 **Compilation** 来获取类型推断分析所需的 **SemanticModel**。你将最后完成此步骤。在此期间，声明一个代表你的测试 Compilation 的占位符变量：
```VB.NET
        Dim test As Compilation = CreateTestCompilation()
```

17) 短暂延迟后，你应该会看到一个错误波浪线出现，报告不存在 **CreateTestCompilation** 方法。按 **Ctrl+Period** 打开智能标签，然后选择**生成方法 Module1.CreateTestCompilation'** 选项。这将在 **Module1** 中为 **CreateTestCompilation** 方法生成一个方法存根。你稍后将回来填写它：
 ![VB 从用法生成方法](images/walkthrough-vb-syntax-transformation-figure1.png)

18) 接下来，编写以下代码以遍历测试 **Compilation** 中的每个 **SyntaxTree**。对于每个 SyntaxTree，用该树的 **SemanticModel** 初始化一个新的 **TypeInferenceRewriter**：
```VB.NET
        For Each sourceTree As SyntaxTree In test.SyntaxTrees
 
            Dim model As SemanticModel = test.GetSemanticModel(sourceTree)
 
            Dim rewriter As New TypeInferenceRewriter(model)

        Next
```

19) 最后，在你刚创建的上述 For Each 语句内，添加以下代码对每个源树执行转换，并在进行了任何编辑时有条件地写出新的转换树。请记住，只有当重写器遇到一个或多个可以使用类型推断简化的局部变量声明时，才应修改树：
```VB.NET
            Dim newSource As SyntaxNode = rewriter.Visit(sourceTree.GetRoot())
 
            If newSource IsNot sourceTree.GetRoot() Then
                File.WriteAllText(sourceTree.FilePath, newSource.ToFullString())
            End If
```

20) 你快完成了！只剩最后一步：创建测试 **Compilation**。由于在本演练中你完全没有使用类型推断，它本来会是一个完美的测试案例。不幸的是，从 VB 项目文件创建 Compilation 超出了本演练的范围。但幸运的是，如果你非常仔细地按照说明操作，还有希望。用以下代码替换 **CreateTestCompilation** 方法的内容。它创建了一个恰好与本演练中描述的项目相匹配的测试编译：
```VB.NET

        Dim globalImports As String() =
            {"Microsoft.CodeAnalysis",
             "Microsoft.CodeAnalysis.VisualBasic",
             "Microsoft.CodeAnalysis.VisualBasic.Syntax"}

        Dim options = New VisualBasicCompilationOptions(OutputKind.ConsoleApplication).
                          WithGlobalImports(From s In globalImports
                                            Select GlobalImport.Parse(s))

        Dim module1Tree As SyntaxTree =
                VisualBasicSyntaxTree.ParseText(File.ReadAllText("..\..\Module1.vb"),, "..\..\Module1.vb",)
        Dim rewriterTree As SyntaxTree =
               VisualBasicSyntaxTree.ParseText(File.ReadAllText(
"..\..\TypeInferenceRewriter.vb"),, "..\..\TypeInferenceRewriter.vb",)

        Dim sourceTrees As SyntaxTree() = {module1Tree, rewriterTree}

        Dim mscorlib As MetadataReference =
            MetadataReference.CreateFromFile(GetType(Object).Assembly.Location)
        Dim codeAnalysis As MetadataReference =
            MetadataReference.CreateFromFile(GetType(SyntaxTree).Assembly.Location)
        Dim vbCodeAnalysis As MetadataReference =
            MetadataReference.CreateFromFile(GetType(VisualBasicSyntaxTree).Assembly.Location)

        Dim references As MetadataReference() = {mscorlib, codeAnalysis, vbCodeAnalysis}

        Return VisualBasicCompilation.Create("TransformationVB",
                                             sourceTrees,
                                             references,
                                             options)
```

21) 你的 **Module1.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Module Module1

    Sub Main()

        Dim test As Compilation = CreateTestCompilation()

        For Each sourceTree As SyntaxTree In test.SyntaxTrees

            Dim model As SemanticModel = test.GetSemanticModel(sourceTree)

            Dim rewriter As New TypeInferenceRewriter(model)

            Dim newSource As SyntaxNode = rewriter.Visit(sourceTree.GetRoot())

            If newSource IsNot sourceTree.GetRoot() Then
                File.WriteAllText(sourceTree.FilePath, newSource.ToFullString())
            End If
        Next

    End Sub

    Private Function CreateTestCompilation() As Compilation

        Dim globalImports As String() =
            {"Microsoft.CodeAnalysis",
             "Microsoft.CodeAnalysis.VisualBasic",
             "Microsoft.CodeAnalysis.VisualBasic.Syntax"}

        Dim options = New VisualBasicCompilationOptions(OutputKind.ConsoleApplication).
                          WithGlobalImports(From s In globalImports
                                            Select GlobalImport.Parse(s))

        Dim module1Tree As SyntaxTree =
                VisualBasicSyntaxTree.ParseText(File.ReadAllText("..\..\Module1.vb"),, "..\..\Module1.vb",)
        Dim rewriterTree As SyntaxTree =
               VisualBasicSyntaxTree.ParseText(File.ReadAllText(
"..\..\TypeInferenceRewriter.vb"),, "..\..\TypeInferenceRewriter.vb",)

        Dim sourceTrees As SyntaxTree() = {module1Tree, rewriterTree}

        Dim mscorlib As MetadataReference =
            MetadataReference.CreateFromFile(GetType(Object).Assembly.Location)
        Dim codeAnalysis As MetadataReference =
            MetadataReference.CreateFromFile(GetType(SyntaxTree).Assembly.Location)
        Dim vbCodeAnalysis As MetadataReference =
            MetadataReference.CreateFromFile(GetType(VisualBasicSyntaxTree).Assembly.Location)

        Dim references As MetadataReference() = {mscorlib, codeAnalysis, vbCodeAnalysis}

        Return VisualBasicCompilation.Create("TransformationVB",
                                             sourceTrees,
                                             references,
                                             options)

    End Function

End Module
```

22) 祈祷好运，运行项目。 
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

23) Visual Studio 应会提示你项目中的文件已更改。点击"全部确定"以重新加载修改后的文件。检查它们，感受一下你的成就 :)
  * 注意没有那些显式且冗余的类型说明符后，代码看起来清晰多了。 

24) 恭喜！你刚刚使用了 **Compiler API** 编写了你自己的重构，它搜索项目中的所有文件以查找特定的语法模式，分析匹配这些模式的源代码的语义，并对其进行转换。你现在正式成为重构大师！
