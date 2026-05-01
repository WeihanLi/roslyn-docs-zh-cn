## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)
* [C# 语法分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-C%23-Syntax-Analysis.md)

## 简介
如今，Visual Basic 和 C# 编译器是黑盒子——文本进入，字节输出——对编译管道中间阶段没有任何透明度。借助 **.NET Compiler Platform**（前身为"Roslyn"），工具和开发人员可以利用编译器用于分析和理解代码的完全相同的数据结构和算法，并确信这些信息是准确且完整的。

在本演练中，我们将探索 **Symbol** 和 **Binding API**。**Syntax API** 暴露了解析器、语法树本身以及用于推理和构建语法树的实用工具。

## 理解编译和符号
**Syntax API** 允许你查看程序的_结构_。然而，你通常会希望获得关于程序语义或_含义_的更丰富信息。虽然可以单独对松散的代码文件或 VB 或 C# 代码片段进行语法分析，但在真空中询问"这个变量的类型是什么"并没有太大意义。类型名称的含义可能依赖于程序集引用、命名空间导入或其他代码文件。这就是 **Compilation** 类发挥作用的地方。

**Compilation** 类似于编译器所看到的单个项目，代表编译 Visual Basic 或 C# 程序所需的一切，例如程序集引用、编译器选项以及要编译的源文件集合。有了这个上下文，你可以推理代码的含义。**Compilations** 允许你找到 **Symbols**——诸如类型、命名空间、成员和变量等实体，名称和其他表达式所引用的对象。将名称和表达式与 **Symbols** 关联的过程称为**绑定**。

与 **SyntaxTree** 一样，**Compilation** 是一个抽象类，具有特定于语言的派生类。创建 Compilation 实例时，必须在 **CSharpCompilation**（或 **VisualBasicCompilation**）类上调用工厂方法。

#### 示例 - 创建编译
此示例展示如何通过添加程序集引用和源文件来创建 **Compilation**。与语法树一样，Symbols API 和 Binding API 中的所有内容都是不可变的。

1) 创建一个新的 C# **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual C# -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**SemanticsCS**"并点击确定。

2) 将 **Program.cs** 的内容替换为以下内容：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
 
namespace SemanticsCS
{
    class Program
    {
        static void Main(string[] args)
        {
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@"using System;
using System.Collections.Generic;
using System.Text;
 
namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(""Hello, World!"");
        }
    }
}");
 
            var root = (CompilationUnitSyntax)tree.GetRoot();
         }
    }
}
```

3) 接下来，将以下代码添加到 **Main** 方法末尾，以构建 **CSharpCompilation** 对象：
```C#
            var compilation = CSharpCompilation.Create("HelloWorld")
                                               .AddReferences(
                                                    MetadataReference.CreateFromFile(
                                                        typeof(object).Assembly.Location))
                                               .AddSyntaxTrees(tree);
```

4) 将光标移至 **Main** 方法的**右花括号**所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

5) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

6) 通过将鼠标悬停在调试器中的 root 变量上并展开数据提示来检查它。

## SemanticModel
一旦你有了 **Compilation**，就可以向它请求该 **Compilation** 中包含的任意 **SyntaxTree** 的 **SemanticModel**。可以查询 **SemanticModels** 来回答诸如"在这个位置有哪些名称在作用域内？""从这个方法可以访问哪些成员？""在这段文本块中使用了哪些变量？"以及"这个名称/表达式指的是什么？"等问题。

#### 示例 - 绑定名称
此示例展示如何为 HelloWorld **SyntaxTree** 获取 **SemanticModel** 对象。获得模型后，第一个 **using** 指令中的名称被绑定以检索 **System** 命名空间的 **Symbol**。

1) 将以下代码添加到 Main 方法末尾。该代码为 HelloWorld **SyntaxTree** 获取 **SemanticModel** 并将其存储在新变量中：
```C#
            var model = compilation.GetSemanticModel(tree);
```

2) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

3) 现在添加以下代码，使用 **SemanticModel.GetSymbolInfo** 方法绑定"**using System;**"指令的 **Name**：
```C#
            var nameInfo = model.GetSymbolInfo(root.Usings[0].Name);
```

4) 执行此语句，将鼠标悬停在 **nameInfo** 变量上并展开数据提示以检查返回的 **SymbolInfo** 对象。
* 注意 **Symbol** 属性。此属性返回此表达式所引用的 **Symbol**。对于不引用任何内容的表达式（例如数字字面量），此属性将为 null。
  * 注意 **Symbol.Kind** 属性返回值 **SymbolKind.Namespace**。

5) 将符号转换为 **NamespaceSymbol** 实例并存储在新变量中：
```C#
            var systemSymbol = (INamespaceSymbol)nameInfo.Symbol;
```

6) 执行此语句，并使用调试器数据提示检查 **systemSymbol** 变量。

7) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop** 调试。

8) 添加以下代码以枚举 **System** 命名空间的子命名空间并将其名称打印到 **Console**：
```C#
            foreach (var ns in systemSymbol.GetNamespaceMembers())
            {
                Console.WriteLine(ns.Name);
            }
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
上一个示例展示了如何绑定名称以找到 **Symbol**。然而，C# 程序中还有其他可以绑定但不是名称的表达式。此示例展示了绑定如何与其他表达式类型一起工作——在本例中是一个简单的字符串字面量。

1) 添加以下代码以在 **SyntaxTree** 中定位"**Hello, World!**"字符串字面量并将其存储在变量中（在此示例中它应该是唯一的 **LiteralExpressionSyntax**）：
```C#
            var helloWorldString = root.DescendantNodes()
                                       .OfType<LiteralExpressionSyntax>()
                                       .First();
```

2) 开始调试程序。

3) 添加以下代码以获取此表达式的 **TypeInfo**：
```C#
            var literalInfo = model.GetTypeInfo(helloWorldString);
```

4) 执行此语句并检查 **literalInfo**。
  * 注意其 **Type** 属性不为 null，并返回 **System.String** 类型的 **INamedTypeSymbol**，因为字符串字面量表达式具有编译时类型 **System.String**

5) 停止程序。

6) 添加以下代码以枚举 **System.String** 类中返回字符串的公共方法，并将其名称打印到 **Console**：
```C#
            var stringTypeSymbol = (INamedTypeSymbol)literalInfo.Type;
 
            Console.Clear();
            foreach (var name in (from method in stringTypeSymbol.GetMembers()
                                                              .OfType<IMethodSymbol>()
                               where method.ReturnType.Equals(stringTypeSymbol) &&
                                     method.DeclaredAccessibility == 
                                                Accessibility.Public
                               select method.Name).Distinct())
            {
                Console.WriteLine(name);
            }
```

7) 按 **Ctrl+F5** 运行程序（不调试）。你应该会看到以下输出：
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

8) 你的 **Program.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;

namespace SemanticsCS
{
    class Program
    {
        static void Main(string[] args)
        {
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@" using System;
using System.Collections.Generic;
using System.Text;
 
namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(""Hello, World!"");
        }
    }
}");

            var root = (CompilationUnitSyntax)tree.GetRoot();

            var compilation = CSharpCompilation.Create("HelloWorld")
                                               .AddReferences(
                                                    MetadataReference.CreateFromFile(
                                                        typeof(object).Assembly.Location))
                                               .AddSyntaxTrees(tree);

            var model = compilation.GetSemanticModel(tree);

            var nameInfo = model.GetSymbolInfo(root.Usings[0].Name);

            var systemSymbol = (INamespaceSymbol)nameInfo.Symbol;

            foreach (var ns in systemSymbol.GetNamespaceMembers())
            {
                Console.WriteLine(ns.Name);
            }

            var helloWorldString = root.DescendantNodes()
                                       .OfType<LiteralExpressionSyntax>()
                                       .First();

            var literalInfo = model.GetTypeInfo(helloWorldString);

            var stringTypeSymbol = (INamedTypeSymbol)literalInfo.Type;

            Console.Clear();
            foreach (var name in (from method in stringTypeSymbol.GetMembers()
                                                              .OfType<IMethodSymbol>()
                                  where method.ReturnType.Equals(stringTypeSymbol) &&
                                        method.DeclaredAccessibility ==
                                                   Accessibility.Public
                                  select method.Name).Distinct())
            {
                Console.WriteLine(name);
            }
        }
    }
}
```

9) 恭喜！你刚刚使用了 **Symbol** 和 **Binding API** 来分析 C# 程序中名称和表达式的含义。
