## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)
* [C# 语法分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-C%23-Syntax-Analysis.md)
* [C# 语义分析入门](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Getting-Started-C%23-Semantic-Analysis.md)

## 简介
本演练基于**入门：语法分析**和**入门：语义分析**演练中探索的概念和技术。如果你尚未完成这些演练，强烈建议在开始本演练之前先完成它们。

在本演练中，你将探索创建和转换语法树的技术。结合你在之前入门演练中学到的技术，你将创建你的第一个命令行重构！

## 不可变性与 .NET Compiler Platform
.NET Compiler Platform 的一个基本原则是不可变性。因为不可变的数据结构在创建后不能被更改，所以它们可以被多个使用者同时安全地共享和分析，而不会有一个工具以不可预测的方式影响另一个工具的危险。无需锁或其他并发措施。这适用于语法树、编译、符号、语义模型以及你将遇到的每个其他数据结构。修改不是直接在原对象上进行的，而是根据对旧对象的指定差异创建新对象。你将把这个概念应用到语法树上来创建树转换！

## 创建和"修改"树
### 使用工厂方法创建节点
要创建 **SyntaxNodes**，必须使用 **SyntaxFactory** 类的工厂方法。对于每种节点、标记或 trivia，都有一个工厂方法可用于创建该类型的实例。通过以自底向上的方式按层次组合节点，你可以创建语法树。 

#### 示例 - 使用工厂方法创建 SyntaxNode
此示例使用 **SyntaxFactory** 类的方法构建表示 **System.Collections.Generic** 命名空间的 **NameSyntax**。

**NameSyntax** 是 C# 中出现的四种名称类型的基类： 

* **IdentifierNameSyntax** 代表简单的单标识符名称，如 **System** 和 **Microsoft**
* **GenericNameSyntax** 代表泛型类型或方法名称，如 **List<int>**
* **QualifiedNameSyntax** 代表 ```<左名称>.<右标识符或泛型名称>``` 形式的限定名称，如 **System.IO**
* **AliasQualifiedNameSyntax** 代表使用程序集外部别名的名称，如 **LibraryV2::Foo**
通过将这些名称组合在一起，你可以创建 C# 语言中可以出现的任何名称。 

1) 创建一个新的 C# **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual C# -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**ConstructionCS**"并点击确定。 

2) 将以下 using 指令添加到文件顶部，以导入 **SyntaxFactory** 类的工厂方法，这样我们以后就可以不加限定地使用它们：
```C#
using static Microsoft.CodeAnalysis.CSharp.SyntaxFactory;
```

3) 将光标移至 **Main** 方法的**右花括号**所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

4) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

5) 首先创建一个简单的 **IdentifierNameSyntax**，代表 **System** 命名空间的名称，并将其赋值给变量。当你从这个节点构建 **QualifiedNameSyntax** 时，你将重用此变量，所以将此变量声明为 **NameSyntax** 类型，以允许它存储两种类型的 **SyntaxNode**——**不要**使用类型推断：
```C#
            NameSyntax name = IdentifierName("System");
```

6) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

7) 打开**即时窗口**。
  * 在 Visual Studio 中，选择 **Debug -> Windows -> Immediate**。

8) 使用即时窗口，输入表达式 **name.ToString()** 并按 Enter 求值。你应该会看到字符串"**System**"作为结果。 

9) 接下来，使用此 **name** 节点作为名称的**左**侧，使用 **Collections** 命名空间的新 **IdentifierNameSyntax** 作为 **QualifiedNameSyntax** 的**右**侧，构造一个 **QualifiedNameSyntax**：
```C#
            name = QualifiedName(name, IdentifierName("Collections"));
```

10) 执行此语句，将 **name** 变量设置为新的 **QualifiedNameSyntax** 节点。

11) 使用即时窗口，求值表达式 **name.ToString()**。它应该求值为"**System.Collections**"。

12) 通过为 **Generic** 命名空间构建另一个 **QualifiedNameSyntax** 节点来继续此模式：
```C#
            name = QualifiedName(name, IdentifierName("Generic"));
```

13) 执行此语句，再次使用即时窗口观察 **name.ToString()** 现在求值为完全限定名称"**System.Collections.Generic**"。 

### 使用 With* 和 ReplaceNode 方法修改节点
因为语法树是不可变的，**Syntax API** 没有提供在构造后直接修改现有语法树的机制。然而，**Syntax API** 确实提供了基于对现有树的指定更改来生成新树的方法。从 **SyntaxNode** 派生的每个具体类都定义了 **With*** 方法，你可以使用它们来指定对其子属性的更改。此外，**ReplaceNode** 扩展方法可用于替换子树中的后代节点。如果没有此方法，更新节点还需要手动更新其父节点以指向新创建的子节点，并在整棵树中重复此过程——这个过程称为_重新旋转_树。 

#### 示例 - 使用 With* 和 ReplaceNode 方法进行转换
此示例使用 **WithName** 方法将 **UsingDirectiveSyntax** 节点中的名称替换为上面构造的名称。

1) 继续上面的示例，添加以下代码以解析示例代码文件：
```C#
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@"using System;
using System.Collections;
using System.Linq;
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
```

  * 注意该文件使用 **System.Collections** 命名空间而不是 **System.Collections.Generic** 命名空间。 

2) 执行这些语句。

3) 使用 **UsingDirectiveSyntax.WithName** 方法创建一个新的 **UsingDirectiveSyntax** 节点，用上面创建的名称更新"**System.Collections**"名称：
```C#
            var oldUsing = root.Usings[1];
            var newUsing = oldUsing.WithName(name);
```

4) 使用即时窗口，求值表达式 **root.ToString()**，并观察原始树未被更改以包含此新的更新节点。

5) 添加以下行，使用 **ReplaceNode** 扩展方法创建一个新树，将现有导入替换为更新后的 **newUsing** 节点，并将新树存储在现有 **root** 变量中：
```C#
            root = root.ReplaceNode(oldUsing, newUsing);
```

6) 执行此语句。

7) 使用即时窗口求值表达式 **root.ToString()**，这次观察树现在正确导入了 **System.Collections.Generic** 命名空间。

8) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop debugging**。

9) 你的 **Program.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using static Microsoft.CodeAnalysis.CSharp.SyntaxFactory;
 
namespace ConstructionCS
{
    class Program
    {
        static void Main(string[] args)
        {
            NameSyntax name = IdentifierName("System");
            name = QualifiedName(name, IdentifierName("Collections"));
            name = QualifiedName(name, IdentifierName("Generic"));
 
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@"using System;
using System.Collections;
using System.Linq;
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
 
            var oldUsing = root.Usings[1];
            var newUsing = oldUsing.WithName(name);
 
            root = root.ReplaceNode(oldUsing, newUsing);
        }
    }
}
```

### 使用 SyntaxRewriter 转换树
**With*** 和 **ReplaceNode** 方法提供了转换语法树单个分支的便捷手段。然而，通常可能需要在语法树上协调执行多个转换。**SyntaxRewriter** 类是 **SyntaxVisitor** 的子类，可用于将转换应用于特定类型的 **SyntaxNode**。还可以将一组转换应用于语法树中任何位置出现的多种类型的 **SyntaxNode**。以下示例在一个简单的命令行重构实现中演示了这一点，该重构在可以使用类型推断的任何地方删除局部变量声明中的显式类型。此示例利用了本演练中讨论的技术以及**入门：语法分析**和**入门：语义分析**演练中的技术。 

#### 示例 - 创建 SyntaxRewriter 以转换语法树
1) 创建一个新的 C# **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual C# -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**TransformationCS**"并点击确定。 

2) 在 **Program.cs** 文件顶部插入以下 **using** 指令：
```C#
using System.IO;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp; 
```

3) 向项目添加新类文件。
  * 在 Visual Studio 中，选择 **Project -> Add Class...** 
  * 在"添加新项"对话框中输入 **TypeInferenceRewriter.cs** 作为文件名。

4) 添加以下 using 指令。
```C#
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using static Microsoft.CodeAnalysis.CSharp.SyntaxFactory;
```

5) 使 **TypeInferenceRewriter** 类继承 **CSharpSyntaxRewriter** 类：
```C#
    public class TypeInferenceRewriter : CSharpSyntaxRewriter
    {
```

6) 添加以下代码，声明一个私有只读字段来保存 **SemanticModel**，并从构造函数中初始化它。你之后将需要此字段来确定可以在哪里使用类型推断：
```C#
        private readonly SemanticModel SemanticModel;
 
        public TypeInferenceRewriter(SemanticModel semanticModel)
        {
            this.SemanticModel = semanticModel;
        }
```

7) 重写 **VisitLocalDeclarationStatement** 方法：
```C#
        public override SyntaxNode VisitLocalDeclarationStatement(
                                       LocalDeclarationStatementSyntax node)
        {

        }
```

  * 注意 **VisitLocalDeclarationStatement** 方法返回 **SyntaxNode**，而不是 **LocalDeclarationStatementSyntax**。在此示例中，你将基于现有节点返回另一个 **LocalDeclarationStatementSyntax** 节点。在其他场景中，一种节点可能完全被另一种节点替换——甚至被删除。

8) 出于本示例的目的，你只处理局部变量声明，尽管类型推断也可以用于 **foreach** 循环、**for** 循环、LINQ 表达式和 lambda 表达式。此外，此重写器只转换最简单形式的声明：
```C#
Type variable = expression;
```

C# 中以下形式的变量声明要么与类型推断不兼容，要么留给读者作为练习。

```C#
// Multiple variables in a single declaration.
Type variable1 = expression1, 
     variable2 = expression2; 
// No initializer.
Type variable; 
```

9) 将以下代码添加到 **VisitLocalDeclarationStatement** 方法主体中，以跳过重写这些形式的声明：
```C#
            if (node.Declaration.Variables.Count > 1) 
            {
                return node;
            }
            if (node.Declaration.Variables[0].Initializer == null)
            {
                return node;
            }
```

  * 注意返回未修改的 **node** 参数会导致该节点不发生重写。 

10) 添加这些语句以提取声明中指定的类型名称，并使用 **SemanticModel** 字段对其进行绑定以获取类型符号。
```C#
            VariableDeclaratorSyntax declarator = node.Declaration.Variables.First();
            TypeSyntax variableTypeName = node.Declaration.Type;
            
            ITypeSymbol variableType = 
                           (ITypeSymbol)SemanticModel.GetSymbolInfo(variableTypeName)
                                                     .Symbol;
```

11) 现在，添加此语句以绑定初始化器表达式： 
```C#
            TypeInfo initializerInfo = 
                         SemanticModel.GetTypeInfo(declarator
                                                   .Initializer
                                                   .Value);
```

12) 最后，添加以下 **if** 语句，如果初始化器表达式的类型与指定类型匹配，则用 **var** 关键字替换现有类型名称：
```C#
            if (variableType == initializerInfo.Type)
            {
                TypeSyntax varTypeName = 
                               IdentifierName("var")
                                     .WithLeadingTrivia(
                                          variableTypeName.GetLeadingTrivia())
                                     .WithTrailingTrivia(
                                          variableTypeName.GetTrailingTrivia());
 
                return node.ReplaceNode(variableTypeName, varTypeName);
            }
            else
            {
                return node;
            }
```

  * 注意此条件是必需的，因为如果类型不匹配，声明可能正在将初始化器表达式转换为基类或接口。在这些情况下删除显式类型将改变程序的语义。
  * 注意 **var** 被指定为标识符而不是关键字，因为 **var** 是一个上下文关键字。
  * 注意前导和尾随 trivia（空白）从旧类型名称转移到 **var** 关键字，以保持垂直空白和缩进。
  * 还要注意，使用 **ReplaceNode** 而不是 **With*** 来转换 **LocalDeclarationStatementSyntax** 更简单，因为类型名称实际上是声明语句的孙子节点。 

13) 你的 **TypeInferenceRewriter.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp; 
using Microsoft.CodeAnalysis.CSharp.Syntax;
using static Microsoft.CodeAnalysis.CSharp.SyntaxFactory;
 
namespace TransformationCS
{
    public class TypeInferenceRewriter : CSharpSyntaxRewriter
    {
        private readonly SemanticModel SemanticModel;
 
        public TypeInferenceRewriter(SemanticModel semanticModel)
        {
            this.SemanticModel = semanticModel;
        }
 
        public override SyntaxNode VisitLocalDeclarationStatement(
                                       LocalDeclarationStatementSyntax node)
        {
            if (node.Declaration.Variables.Count > 1) 
            {
                return node;
            }
            if (node.Declaration.Variables[0].Initializer == null)
            {
                return node;
            }
 
            VariableDeclaratorSyntax declarator = node.Declaration.Variables.First();
            TypeSyntax variableTypeName = node.Declaration.Type;
            
            ITypeSymbol variableType = 
                           (ITypeSymbol)SemanticModel.GetSymbolInfo(variableTypeName)
                                                    .Symbol;
            
            TypeInfo initializerInfo = 
                         SemanticModel.GetTypeInfo(declarator
                                                   .Initializer
                                                   .Value);
            
            if (variableType == initializerInfo.Type)
            {
                TypeSyntax varTypeName = 
                               IdentifierName("var")
                                     .WithLeadingTrivia(
                                          variableTypeName.GetLeadingTrivia())
                                     .WithTrailingTrivia(
                                          variableTypeName.GetTrailingTrivia());
 
                return node.ReplaceNode(variableTypeName, varTypeName);
            }
            else
            {
                return node;
            }
        }
    }
}
```

14) 返回 **Program.cs** 文件。

15) 要测试你的 **TypeInferenceRewriter**，你需要创建一个测试 **Compilation** 来获取类型推断分析所需的 **SemanticModels**。你将最后完成此步骤。在此期间，声明一个代表你的测试 Compilation 的占位符变量：
```C#
            Compilation test = CreateTestCompilation();
```

16) 稍等片刻后，你应该会看到一个错误波浪线出现，报告不存在 **CreateTestCompilation** 方法。按 **Ctrl+Period** 打开灯泡，然后按 Enter 调用**生成方法存根**命令。这将在 **Program** 中为 **CreateTestCompilation** 方法生成一个方法存根。你稍后将回来填写它：
![C# 从用法生成方法](images/walkthrough-csharp-syntax-transformation-figure1.png)

17) 接下来，编写以下代码以遍历测试 **Compilation** 中的每个 **SyntaxTree**。对于每个 SyntaxTree，用该树的 **SemanticModel** 初始化一个新的 **TypeInferenceRewriter**：
```C#
            foreach (SyntaxTree sourceTree in test.SyntaxTrees)
            {
                SemanticModel model = test.GetSemanticModel(sourceTree);
 
                TypeInferenceRewriter rewriter = new TypeInferenceRewriter(model);
            }
```

18) 最后，在你刚创建的 **foreach** 语句内，添加以下代码对每个源树执行转换，并在进行了任何编辑时有条件地写出新的转换树。请记住，只有当重写器遇到一个或多个可以使用类型推断简化的局部变量声明时，才应修改树：
```C#
                SyntaxNode newSource = rewriter.Visit(sourceTree.GetRoot());

                if (newSource != sourceTree.GetRoot())
                {
                    File.WriteAllText(sourceTree.FilePath, newSource.ToFullString());
                }
```

19) 你快完成了！只剩最后一步：创建测试 **Compilation**。由于在本演练中你完全没有使用类型推断，它本来会是一个完美的测试案例。不幸的是，从 C# 项目文件创建 Compilation 超出了本演练的范围。但幸运的是，如果你非常仔细地按照说明操作，还有希望。用以下代码替换 **CreateTestCompilation** 方法的内容。它创建了一个恰好与本演练中描述的项目相匹配的测试编译：
```C#
            String programPath = @"..\..\Program.cs";
            String programText = File.ReadAllText(programPath);
            SyntaxTree programTree =
                           CSharpSyntaxTree.ParseText(programText)
                                           .WithFilePath(programPath);

            String rewriterPath = @"..\..\TypeInferenceRewriter.cs";
            String rewriterText = File.ReadAllText(rewriterPath);
            SyntaxTree rewriterTree =
                           CSharpSyntaxTree.ParseText(rewriterText)
                                           .WithFilePath(rewriterPath);

            SyntaxTree[] sourceTrees = { programTree, rewriterTree };

            MetadataReference mscorlib =
                    MetadataReference.CreateFromFile(typeof(object).Assembly.Location);
            MetadataReference codeAnalysis =
                    MetadataReference.CreateFromFile(typeof(SyntaxTree).Assembly.Location);
            MetadataReference csharpCodeAnalysis =
                    MetadataReference.CreateFromFile(typeof(CSharpSyntaxTree).Assembly.Location);

            MetadataReference[] references = { mscorlib, codeAnalysis, csharpCodeAnalysis };

            return CSharpCompilation.Create("TransformationCS",
                                            sourceTrees,
                                            references,
                                            new CSharpCompilationOptions(
                                                    OutputKind.ConsoleApplication));
```

20) 你的 **Program.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;

namespace TransformationCS
{
    internal class Program
    {
        private static void Main()
        {
            Compilation test = CreateTestCompilation();

            foreach (SyntaxTree sourceTree in test.SyntaxTrees)
            {
                SemanticModel model = test.GetSemanticModel(sourceTree);

                TypeInferenceRewriter rewriter = new TypeInferenceRewriter(model);

                SyntaxNode newSource = rewriter.Visit(sourceTree.GetRoot());

                if (newSource != sourceTree.GetRoot())
                {
                    File.WriteAllText(sourceTree.FilePath, newSource.ToFullString());
                }
            }
        }

        private static Compilation CreateTestCompilation()
        {
            String programPath = @"..\..\Program.cs";
            String programText = File.ReadAllText(programPath);
            SyntaxTree programTree =
                           CSharpSyntaxTree.ParseText(programText)
                                           .WithFilePath(programPath);

            String rewriterPath = @"..\..\TypeInferenceRewriter.cs";
            String rewriterText = File.ReadAllText(rewriterPath);
            SyntaxTree rewriterTree =
                           CSharpSyntaxTree.ParseText(rewriterText)
                                           .WithFilePath(rewriterPath);


            SyntaxTree[] sourceTrees = { programTree, rewriterTree };

            MetadataReference mscorlib =
                    MetadataReference.CreateFromFile(typeof(object).Assembly.Location);
            MetadataReference codeAnalysis =
                    MetadataReference.CreateFromFile(typeof(SyntaxTree).Assembly.Location);
            MetadataReference csharpCodeAnalysis =
                    MetadataReference.CreateFromFile(typeof(CSharpSyntaxTree).Assembly.Location);

            MetadataReference[] references = { mscorlib, codeAnalysis, csharpCodeAnalysis };

            return CSharpCompilation.Create("TransformationCS",
                                            sourceTrees,
                                            references,
                                            new CSharpCompilationOptions(
                                                    OutputKind.ConsoleApplication));
        }
    }
}
```

21) 祈祷好运，运行项目。 
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

22) Visual Studio 应会提示你项目中的文件已更改。点击"**全部确定**"以重新加载修改后的文件。检查它们，感受一下你的成就 :)
  * 注意没有那些显式且冗余的类型说明符后，代码看起来清晰多了。 

23) 恭喜！你刚刚使用了 **Compiler API** 编写了你自己的重构，它搜索 C# 项目中的所有文件以查找特定的语法模式，分析匹配这些模式的源代码的语义，并对其进行转换。你现在正式成为重构大师！
