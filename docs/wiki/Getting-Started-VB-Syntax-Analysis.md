## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)

## 简介
如今，Visual Basic 和 C# 编译器是黑盒子——文本进入，字节输出——对编译管道中间阶段没有任何透明度。借助 **.NET Compiler Platform**（前身为"Roslyn"），工具和开发人员可以利用编译器用于分析和理解代码的完全相同的数据结构和算法，并确信这些信息是准确且完整的。

在本演练中，我们将探索 **Syntax API**。**Syntax API** 暴露了解析器、语法树本身以及用于推理和构建语法树的实用工具。

## 理解语法树
**Syntax API** 暴露了编译器用于理解 Visual Basic 和 C# 程序的语法树。它们由构建项目或开发者按下 F5 时运行的同一解析器生成。语法树与语言完全保真；代码文件中的每一位信息都在树中表示，包括注释或空白字符等内容。将语法树写入文本将重现被解析的完全原始文本。语法树也是不可变的；一旦创建，语法树永远无法更改。这意味着树的使用者可以在多个线程上分析树，无需锁或其他并发措施，并保证数据永远不会更改。

语法树的四个主要构建块是：

* **SyntaxTree** 类，其实例代表整个解析树。**SyntaxTree** 是一个抽象类，具有特定于语言的派生类。要解析特定语言中的语法，你需要使用 **VisualBasicSyntaxTree**（或 **CSharpSyntaxTree**）类上的解析方法。
* **SyntaxNode** 类，其实例代表声明、语句、子句和表达式等语法构造。
* **SyntaxToken** 结构，代表单个关键字、标识符、运算符或标点符号。
* 最后是 **SyntaxTrivia** 结构，代表语法上无关紧要的信息位，例如标记之间的空白、预处理指令和注释。
**SyntaxNodes** 按层次组成一棵树，完整地表示 Visual Basic 或 C# 代码片段中的所有内容。例如，如果你使用 Syntax Visualizer（在 Visual Studio 中，选择 **View -> Other Windows -> Syntax Visualizer**）检查以下 Visual Basic 源文件，其树视图将如下所示： 

**SyntaxNode**：蓝色 | **SyntaxToken**：绿色 | **SyntaxTrivia**：红色
![Visual Basic 代码文件](images/walkthrough-vb-syntax-figure1.png)

通过导航此树结构，你可以找到代码文件中的任何语句、表达式、标记或空白字符！

## 遍历树
### 手动遍历
以下步骤使用**编辑并继续**演示如何解析 VB 源文本并找到源中包含的参数声明。

#### 示例 - 手动遍历树
1) 创建一个新的 Visual Basic **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual Basic -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**GettingStartedVB**"并点击确定。

2) 在 **Module1.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off
```

  * 某些读者可能在项目级别默认启用了 **Option Strict**。在本演练中将 **Option Strict** **关闭**可以通过减少大量类型转换需求来简化许多示例。

3) 将以下代码输入到 **Main** 方法中：
```VB.NET
        Dim tree As SyntaxTree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections
Imports System.Linq
Imports System.Text

Namespace HelloWorld
    Module Program
        Sub Main(args As String())
           Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")

        Dim root As Syntax.CompilationUnitSyntax = tree.GetRoot()
```

4) 将光标移至 **Main** 方法的 **End Sub** 所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

5) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

6) 通过将鼠标悬停在调试器中的 root 变量上并展开数据提示来检查它。
  * 注意其 **Imports** 属性是一个包含四个元素的集合；解析文本中每个 Import 语句各对应一个。
  * 注意根节点的 **KindText** 是 **CompilationUnit**。
  * 注意 **CompilationUnitSyntax** 节点的 **Members** 集合包含一个元素。

7) 在 Main 方法末尾插入以下语句，将根 **CompilationUnitSyntax** 变量的第一个成员存储在新变量中：
```VB.NET
        Dim firstMember = root.Members(0)
```

8) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

9) 将鼠标悬停在 **firstMember** 变量上并展开数据提示以检查它。 
  * 注意其 **KindText** 是 **NamespaceBlock**。
  * 注意其运行时类型实际上是 **NamespaceBlockSyntax**。 

10) 将此节点转换为 **NamespaceBlockSyntax** 并存储在新变量中：
```VB.NET
        Dim helloWorldDeclaration As Syntax.NamespaceBlockSyntax = firstMember
```

11) 执行此语句并检查 **helloWorldDeclaration** 变量。
  * 注意与 **CompilationUnitSyntax** 一样，**NamespaceBlockSyntax** 也有 **Members** 集合。

12) 检查 **Members** 集合。
  * 注意它包含一个成员。检查它。
    * 注意其 **KindText** 是 **ModuleBlock**。
    * 注意其运行时类型是 **ModuleBlockSyntax**。

13) 将此节点转换为 **ModuleBlockSyntax** 并存储在新变量中：
```VB.NET
        Dim programDeclaration As Syntax.ModuleBlockSyntax = 
                helloWorldDeclaration.Members(0)
```

14) 执行此语句。

15) 在 **programDeclaration.Members** 集合中找到 **Main** 声明并存储在新变量中：
```VB.NET
        Dim mainDeclaration As Syntax.MethodBlockSyntax = programDeclaration.Members(0)
```

16) 执行此语句并检查 **MethodBlockSyntax** 对象的成员。
  * 检查 **BlockStatement** 属性。
    * 注意 **AsClause** 和 **Identifier** 属性。
    * 注意 **ParameterList** 属性；检查它。
      * 注意它包含参数列表的左括号和右括号，以及参数列表本身。
      * 注意参数以 **SeparatedSyntaxList**(**Of** **ParameterSyntax**) 的形式存储。
  * 注意 **Statements** 属性。

17) 将 **Main** 声明的第一个参数存储在变量中。 
```VB.NET
        Dim argsParameter As Syntax.ParameterSyntax = 
                mainDeclaration.BlockStatement.ParameterList.Parameters(0)
```

18) 执行此语句并检查 **argsParameter** 变量。
  * 检查 **Identifier** 属性；注意它的类型是 **ModifiedIdentifierSyntax**。此类型表示带有可选可空修饰符（**x?**）和/或数组秩说明符（**arr(,)**）的普通标识符。 
  * 注意 **ModifiedIdentifierSyntax** 有一个结构类型 **SyntaxToken** 的 **Identifier** 属性。
  * 检查 **Identifier** **SyntaxToken** 的属性；注意标识符的文本可以在 **ValueText** 属性中找到。

19) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop Debugging**。

20) 你的程序现在应如下所示：
```VB.NET
Option Strict Off

Module Module1
 
    Sub Main()
 
        Dim tree As SyntaxTree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections
Imports System.Linq
Imports System.Text

Namespace HelloWorld
    Module Program
        Sub Main(args As String())
           Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")

        Dim root As Syntax.CompilationUnitSyntax = tree.GetRoot()
 
        Dim firstMember = root.Members(0)

        Dim helloWorldDeclaration As Syntax.NamespaceBlockSyntax = firstMember
 
        Dim programDeclaration As Syntax.ModuleBlockSyntax = 
                helloWorldDeclaration.Members(0)
 
        Dim mainDeclaration As Syntax.MethodBlockSyntax = programDeclaration.Members(0)
 
        Dim argsParameter As Syntax.ParameterSyntax = 
                mainDeclaration.BlockStatement.ParameterList.Parameters(0)
 
    End Sub
 
End Module
```

### 查询方法
除了使用 **SyntaxNode** 派生类的属性遍历树之外，还可以使用 **SyntaxNode** 上定义的查询方法探索语法树。熟悉 XPath 的人应该对这些方法非常熟悉。你可以将这些方法与 LINQ 结合使用，快速在树中查找内容。 

#### 示例 - 使用查询方法
1) 使用 IntelliSense，通过 root 变量检查 **SyntaxNode** 类的成员。
  * 注意查询方法，如 **DescendantNodes**、**AncestorsAndSelf** 和 **ChildNodes**。

2) 将以下语句添加到 **Main** 方法末尾。第一条语句使用 LINQ 表达式和 **DescendantNodes** 方法定位与上一示例中相同的参数：
```VB.NET
Dim firstParameters = From methodStatement In root.DescendantNodes().
                                                   OfType(Of Syntax.MethodStatementSyntax)()
                      Where methodStatement.Identifier.ValueText = "Main"
                      Select methodStatement.ParameterList.Parameters.First()
 
Dim argsParameter2 = firstParameters.First()
```

3) 开始调试程序。

4) 打开即时窗口。
  * 在 Visual Studio 中，选择 **Debug -> Windows -> Immediate**。

5) 使用即时窗口，输入表达式 **? argsParameter Is argsParameter2** 并按 Enter 求值。 
  * 注意 LINQ 表达式找到了与手动导航树时相同的参数。

6) 停止程序。

### SyntaxWalker
你通常会希望在语法树中找到特定类型的所有节点，例如文件中的每个属性声明。通过继承 **VisualBasicSyntaxWalker** 类并重写 **VisitPropertyStatement** 方法，你可以在不事先了解语法树结构的情况下处理语法树中的每个属性声明。**VisualBasicSyntaxWalker** 是 **SyntaxVisitor** 的一种特定类型，它递归地访问节点及其每个子节点。

#### 示例 - 实现 VisualBasicSyntaxWalker
此示例展示如何实现一个 **VisualBasicSyntaxWalker**，它检查整个语法树并收集它找到的任何未导入 **System** 命名空间的 **Imports** 语句。

1) 创建一个新的 Visual Basic **Stand-Alone Code Analysis Tool** 项目；将其命名为"**ImportsCollectorVB**"。

3) 在 **Module1.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off
```

3) 将以下代码输入到 **Main** 方法中：
```VB.NET
        Dim tree As SyntaxTree = VisualBasicSyntaxTree.ParseText(
"Imports Microsoft.VisualBasic
Imports System
Imports System.Collections
Imports Microsoft.Win32
Imports System.Linq
Imports System.Text
Imports Microsoft.CodeAnalysis
Imports System.ComponentModel
Imports System.Runtime.CompilerServices
Imports Microsoft.CodeAnalysis.VisualBasic

Namespace HelloWorld
    Module Program
        Sub Main(args As String())
           Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")

        Dim root As Syntax.CompilationUnitSyntax = tree.GetRoot()
```

4) 注意此源文本包含一长串 **Imports** 语句。

5) 向项目添加新类文件。
  * 在 Visual Studio 中，选择 **Project -> Add Class...** 
  * 在"添加新项"对话框中输入 **ImportsCollector.vb** 作为文件名。

6) 在 **ImportsCollector.vb** 文件顶部输入以下行：
```VB.NET
Option Strict Off
```

7) 使此文件中的新 **ImportsCollector** 类继承 **VisualBasicSyntaxWalker** 类：
```VB.NET
Public Class ImportsCollector
    Inherits VisualBasicSyntaxWalker
```

8) 在 **ImportsCollector** 类中声明一个公共只读字段；我们将使用此变量存储找到的 **ImportsStatementSyntax** 节点：
```VB.NET
    Public ReadOnly [Imports] As New List(Of Syntax.ImportsStatementSyntax)()
```

9) 重写 **VisitSimpleImportsClause** 方法：
```VB.NET
    Public Overrides Sub VisitSimpleImportsClause(
                             node As SimpleImportsClauseSyntax
                         )
 
    End Sub
```

10) 使用 IntelliSense，通过此方法的 **node** 参数检查 **SimpleImportsClauseSyntax** 类。
  * 注意 **NameSyntax** 类型的 **Name** 属性；它存储被导入命名空间的名称。

11) 将 **VisitSimpleImportsClause** 方法中的代码替换为以下内容，如果 **Name** 不引用 **System** 命名空间或其任何子命名空间，则有条件地将找到的 **node** 添加到 **[Imports]** 集合中：
```VB.NET
        If node.Name.ToString() = "System" OrElse
           node.Name.ToString().StartsWith("System.") Then Return
 
        [Imports].Add(node.Parent)
```

12) **ImportsCollector.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Public Class ImportsCollector
    Inherits VisualBasicSyntaxWalker
 
    Public ReadOnly [Imports] As New List(Of Syntax.ImportsStatementSyntax)()
 
    Public Overrides Sub VisitSimpleImportsClause(
                             node As SimpleImportsClauseSyntax
                         )
 
        If node.Name.ToString() = "System" OrElse
           node.Name.ToString().StartsWith("System.") Then Return
 
        [Imports].Add(node.Parent)
 
    End Sub
End Class
```

13) 返回 **Module1.vb** 文件。

14) 将以下代码添加到 **Main** 方法末尾，以创建 **ImportsCollector** 的实例，使用该实例访问解析树的根，并遍历收集到的 **ImportsStatementSyntax** 节点，将其名称打印到 **Console**：
```VB.NET
        Dim visitor As New ImportsCollector()
        visitor.Visit(root)
 
        For Each statement In visitor.Imports
            Console.WriteLine(statement)
        Next
```

15) 你的 **Module1.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Module Module1
 
    Sub Main()
 
        Dim tree As SyntaxTree = VisualBasicSyntaxTree.ParseText(
"Imports Microsoft.VisualBasic
Imports System
Imports System.Collections
Imports Microsoft.Win32
Imports System.Linq
Imports System.Text
Imports Microsoft.CodeAnalysis
Imports System.ComponentModel
Imports System.Runtime.CompilerServices
Imports Microsoft.CodeAnalysis.VisualBasic
 
Namespace HelloWorld
    Module Program
        Sub Main(args As String())
           Console.WriteLine(""Hello, World!"")
        End Sub
    End Module
End Namespace")

        Dim root As Syntax.CompilationUnitSyntax = tree.GetRoot()
 
        Dim visitor As New ImportsCollector()
        visitor.Visit(root)
 
        For Each statement In visitor.Imports
            Console.WriteLine(statement)
        Next
 
    End Sub
 
End Module
```

16) 按 **Ctrl+F5** 运行程序（不调试）。你应该会看到以下输出：
```
Imports Microsoft.VisualBasic
Imports Microsoft.Win32
Imports Microsoft.CodeAnalysis
Imports Microsoft.CodeAnalysis.VisualBasic
Press any key to continue . . .
```

17) 观察到 walker 已找到所有四个非 **System** 命名空间的 **Imports** 语句。

18) 恭喜！你刚刚使用了 **Syntax API** 在 VB 源代码中定位特定类型的 VB 语句和声明。
