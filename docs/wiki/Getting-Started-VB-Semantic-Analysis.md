
## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)
* [VB 语法分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-VB-Syntax-Analysis.md)

## 简介
如今，Visual Basic 和 C# 编译器是黑盒子——文本进入，字节输出——对编译管道中间阶段没有任何透明度。借助 **.NET Compiler Platform**（前身为"Roslyn"），工具和开发人员可以利用编译器用于分析和理解代码的完全相同的数据结构和算法，并确信这些信息是准确且完整的。

在本演练中，我们将探索 **Symbol** 和 **Binding API**。**Syntax API** 暴露了解析器、语法树本身以及用于推理和构建语法树的实用工具。

## 理解编译和符号
 **Syntax API** 允许你查看程序的_结构_。然而，你通常会希望获得关于程序语义或_含义_的更丰富信息。虽然可以单独对松散的代码文件或 VB 或 C# 代码片段进行语法分析，但在真空中询问"这个变量的类型是什么"并没有太大意义。类型名称的含义可能依赖于程序集引用、命名空间导入或其他代码文件。这就是 **Compilation** 类发挥作用的地方。

**Compilation** 类似于编译器所看到的单个项目，代表编译 Visual Basic 或 C# 程序所需的一切，例如程序集引用、编译器选项以及要编译的源文件集合。有了这个上下文，你可以推理代码的含义。**Compilations** 允许你找到 **Symbols**——诸如类型、命名空间、成员和变量等实体，名称和其他表达式所引用的对象。将名称和表达式与 **Symbols** 关联的过程称为**绑定**。

与 **SyntaxTree** 一样，**Compilation** 是一个抽象类，具有特定于语言的派生类。创建 Compilation 实例时，必须在 **VisualBasicCompilation**（或 **CSharpCompilation**）类上调用工厂方法。

#### 示例 - 创建编译
此示例展示如何通过添加程序集引用和源文件来创建 **Compilation**。与语法树一样，Symbols API 和 Binding API 中的所有内容都是不可变的。

1) 创建一个新的 Visual Basic **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual Basic -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**SemanticsVB**"并点击确定。 

2) 将 **Module1.vb** 的内容替换为以下内容：
```VB.NET
Option Strict Off

Module Module1
 
    Sub Main()
 
        Dim tree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections.Generic
Imports System.Text
 
Namespace HelloWorld
    Class Class1
        Shared Sub Main(args As String())
            Console.WriteLine(""Hello, World!"")
        End Sub
    End Class
End Namespace")
 
        Dim root As CompilationUnitSyntax = tree.GetRoot()
    End Sub
 
End Module
```

  * 某些读者可能在项目级别默认启用了 **Option Strict**。在本演练中将 **Option Strict** **关闭**可以通过减少大量类型转换需求来简化许多示例。

3) 接下来，将以下代码添加到 **Main** 方法末尾，以构建 **Compilation** 对象：
```VB.NET
        Dim compilation As Compilation =
                VisualBasicCompilation.Create("HelloWorld").
                                       AddReferences(MetadataReference.CreateFromAssembly(
                                                         GetType(Object).Assembly)).
                                       AddSyntaxTrees(tree)
```

4) 将光标移至 **Main** 方法的 **End Sub** 所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

5) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

6) 将鼠标悬停在 compilation 变量上并展开数据提示，在调试器中检查 **Compilation** 对象。

## SemanticModel
一旦你有了 **Compilation**，就可以向它请求该 **Compilation** 中包含的任意 **SyntaxTree** 的 **SemanticModel**。可以查询 **SemanticModels** 来回答诸如"在这个位置有哪些名称在作用域内？""从这个方法可以访问哪些成员？""在这段文本块中使用了哪些变量？"以及"这个名称/表达式指的是什么？"等问题。

#### 示例 - 绑定名称
此示例展示如何为 HelloWorld **SyntaxTree** 获取 **SemanticModel** 对象。获得模型后，第一个 **Imports** 语句中的名称被绑定以检索 **System** 命名空间的 **Symbol**。

1) 将以下代码添加到 Main 方法末尾。该代码为 HelloWorld **SyntaxTree** 获取 **SemanticModel** 并将其存储在新变量中：
```VB.NET
        Dim model = compilation.GetSemanticModel(tree)
```

2) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

3) 现在添加以下代码，使用 **SemanticModel.GetSymbolInfo** 方法绑定"**Imports System**"语句的 **Name**：
```VB.NET
        Dim firstImport As SimpleImportsClauseSyntax =
                root.Imports(0).ImportsClauses(0)
 
        Dim nameInfo = model.GetSymbolInfo(firstImport.Name)
```

4) 执行这些语句，将鼠标悬停在 **nameInfo** 变量上并展开数据提示以检查返回的 **SymbolInfo** 对象。
* 注意 **Symbol** 属性。此属性返回此表达式所引用的 **Symbol**。对于不引用任何内容的表达式（例如数字字面量），此属性将为 null。
  * 注意 **Symbol.Kind** 属性返回值 **SymbolKind.Namespace**。

5) 将符号转换为 **NamespaceSymbol** 实例并存储在新变量中：
```VB.NET
        Dim systemSymbol As INamespaceSymbol = nameInfo.Symbol
```

6) 执行此语句，并使用调试器数据提示检查 **systemSymbol** 变量。
 
7) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop Debugging**。

8) 添加以下代码以枚举 **System** 命名空间的子命名空间并将其名称打印到 **Console**：
```VB.NET
        For Each ns In systemSymbol.GetNamespaceMembers()
            Console.WriteLine(ns.Name)
        Next
```

9) 按 **Ctrl+F5** 运行程序。你应该会看到以下输出：
```
Collections
Configuration
Deployment
Diagnostics
Globalization
IO
Reflection
Resources
Runtime
Security
StubHelpers
Text
Threading
Press any key to continue . . .
```

#### 示例 - 绑定表达式
上一个示例展示了如何绑定名称以找到 **Symbol**。然而，VB 程序中还有其他可以绑定但不是名称的表达式。此示例展示了绑定如何与其他表达式类型一起工作——在本例中是一个简单的字符串字面量。

1) 添加以下代码以在 **SyntaxTree** 中定位"**Hello, World!**"字符串字面量并将其存储在变量中（在此示例中它应该是唯一的 **LiteralExpressionSyntax**）：
```VB.NET
        Dim helloWorldString = root.DescendantNodes().
                                    OfType(Of LiteralExpressionSyntax)().
                                    First()
```

2) 开始调试程序。

3) 添加以下代码以获取此表达式的 **TypeInfo**：
```VB.NET
        Dim literalInfo = model.GetTypeInfo(helloWorldString)
```

4) 执行此语句并检查 **literalInfo**。
  * 注意其 **Type** 属性不为 null，并返回 **System.String** 类型的 **INamedTypeSymbol**，因为字符串字面量表达式具有编译时类型 **System.String**

5) 停止程序。

6) 添加以下代码以枚举 **System.String** 类中返回字符串的公共方法，并将其名称打印到 **Console**：
```VB.NET
        Dim stringTypeSymbol As INamedTypeSymbol = literalInfo.Type

        Dim methodNames = From method In stringTypeSymbol.GetMembers().
                                                          OfType(Of IMethodSymbol)()
                          Where method.ReturnType.Equals(stringTypeSymbol) AndAlso
                                method.DeclaredAccessibility = Accessibility.Public
                          Select method.Name Distinct

        Console.Clear()
        For Each name In methodNames
            Console.WriteLine(name)
        Next
```

7) 按 **Ctrl+F5** 运行程序。你应该会看到以下输出：
```
Join
Substring
Trim
TrimStart
TrimEnd
Normalize
PadLeft
PadRight
ToLower
ToLowerInvariant
ToUpper
ToUpperInvariant
ToString
Insert
Replace
Remove
Format
Copy
Concat
Intern
IsInterned
Press any key to continue . . .
```

8) 你的 **Module1.vb** 文件现在应如下所示：
```VB.NET
Option Strict Off

Module Module1

    Sub Main()

        Dim tree = VisualBasicSyntaxTree.ParseText(
"Imports System
Imports System.Collections.Generic
Imports System.Text
 
Namespace HelloWorld
    Class Class1
        Shared Sub Main(args As String())
            Console.WriteLine(""Hello, World!"")
        End Sub
    End Class
End Namespace")

        Dim root As CompilationUnitSyntax = tree.GetRoot()

        Dim compilation As Compilation =
                VisualBasicCompilation.Create("HelloWorld").
                                       AddReferences(MetadataReference.CreateFromAssembly(
                                                         GetType(Object).Assembly)).
                                       AddSyntaxTrees(tree)

        Dim model = compilation.GetSemanticModel(tree)

        Dim firstImport As SimpleImportsClauseSyntax =
                root.Imports(0).ImportsClauses(0)

        Dim nameInfo = model.GetSymbolInfo(firstImport.Name)

        Dim systemSymbol As INamespaceSymbol = nameInfo.Symbol

        For Each ns In systemSymbol.GetNamespaceMembers()
            Console.WriteLine(ns.Name)
        Next

        Dim helloWorldString = root.DescendantNodes().
                                    OfType(Of LiteralExpressionSyntax)().
                                    First()

        Dim literalInfo = model.GetTypeInfo(helloWorldString)

        Dim stringTypeSymbol As INamedTypeSymbol = literalInfo.Type

        Dim methodNames = From method In stringTypeSymbol.GetMembers().
                                                          OfType(Of IMethodSymbol)()
                          Where method.ReturnType.Equals(stringTypeSymbol) AndAlso
                                method.DeclaredAccessibility = Accessibility.Public
                          Select method.Name Distinct

        Console.Clear()
        For Each name In methodNames
            Console.WriteLine(name)
        Next
    End Sub

End Module
```

9) 恭喜！你刚刚使用了 **Symbol** 和 **Binding API** 来分析 VB 程序中名称和表达式的含义。
