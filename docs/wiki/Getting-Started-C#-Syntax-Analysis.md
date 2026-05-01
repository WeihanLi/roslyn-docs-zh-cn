## 前提条件
* [Visual Studio 2015](https://www.visualstudio.com/downloads)
* [.NET Compiler Platform SDK](https://aka.ms/roslynsdktemplates)

## 简介
如今，Visual Basic 和 C# 编译器是黑盒子——文本进入，字节输出——对编译管道中间阶段没有任何透明度。借助 **.NET Compiler Platform**（前身为"Roslyn"），工具和开发人员可以利用编译器用于分析和理解代码的完全相同的数据结构和算法，并确信这些信息是准确且完整的。

在本演练中，我们将探索 **Syntax API**。**Syntax API** 暴露了解析器、语法树本身以及用于推理和构建语法树的实用工具。

## 理解语法树
**Syntax API** 暴露了编译器用于理解 Visual Basic 和 C# 程序的语法树。它们由构建项目或开发者按下 F5 时运行的同一解析器生成。语法树与语言完全保真；代码文件中的每一位信息都在树中表示，包括注释或空白字符等内容。将语法树写入文本将重现被解析的完全原始文本。语法树也是不可变的；一旦创建，语法树永远无法更改。这意味着树的使用者可以在多个线程上分析树，无需锁或其他并发措施，并保证数据永远不会在其下方更改。

语法树的四个主要构建块是：

* **SyntaxTree** 类，其实例代表整个解析树。**SyntaxTree** 是一个抽象类，具有特定于语言的派生类。要解析特定语言中的语法，你需要使用 **CSharpSyntaxTree**（或 **VisualBasicSyntaxTree**）类上的解析方法。
* **SyntaxNode** 类，其实例代表声明、语句、子句和表达式等语法构造。
* **SyntaxToken** 结构，代表单个关键字、标识符、运算符或标点符号。
* 最后是 **SyntaxTrivia** 结构，代表语法上无关紧要的信息位，例如标记之间的空白、预处理指令和注释。

Trivia、标记和节点按层次组成一棵树，完整地表示 Visual Basic 或 C# 代码片段中的所有内容。例如，如果你使用 **Syntax Visualizer**（在 Visual Studio 中，选择 **View -> Other Windows -> Syntax Visualizer**）检查以下 C# 源文件，其树视图将如下所示：

**SyntaxNode**：蓝色 | **SyntaxToken**：绿色 | **SyntaxTrivia**：红色
![C# 代码文件](images/walkthrough-csharp-syntax-figure1.png)

通过导航此树结构，你可以找到代码文件中的任何语句、表达式、标记或空白字符！

## 遍历树
### 手动遍历
以下步骤使用**编辑并继续**演示如何解析 C# 源文本并找到源中包含的参数声明。

#### 示例 - 手动遍历树
1) 创建一个新的 C# **Stand-Alone Code Analysis Tool** 项目。
  * 在 Visual Studio 中，选择 **File -> New -> Project...** 以显示"新建项目"对话框。
  * 在 **Visual C# -> Extensibility** 下，选择 **Stand-Alone Code Analysis Tool**。
  * 将项目命名为"**GettingStartedCS**"并点击确定。 

2) 将以下 using 指令添加到 Program.cs 文件：
```C#
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
```

3) 将以下代码输入到 **Main** 方法中：
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

4) 将光标移至 **Main** 方法的**右花括号**所在行，并在那里设置断点。
  * 在 Visual Studio 中，选择 **Debug -> Toggle Breakpoint**。

5) 运行程序。
  * 在 Visual Studio 中，选择 **Debug -> Start Debugging**。

6) 通过将鼠标悬停在调试器中的 root 变量上并展开数据提示来检查它。
  * 注意其 **Usings** 属性是一个包含四个元素的集合；解析文本中每个 using 指令各对应一个。
  * 注意根节点的 **KindText** 是 **CompilationUnit**。
  * 注意 **CompilationUnitSyntax** 节点的 **Members** 集合包含一个元素。

7) 在 Main 方法末尾插入以下语句，将根 **CompilationUnitSyntax** 变量的第一个成员存储在新变量中：
```C#
            var firstMember = root.Members[0];
```

8) 将此语句设置为下一条要执行的语句并执行它。
  * 右键单击此行并选择 **Set Next Statement**。
  * 在 Visual Studio 中，选择 **Debug -> Step Over** 以执行此语句并初始化新变量。
  * 对于以下每个步骤，你都需要重复此过程，因为我们会引入新变量并用调试器检查它们。

9) 将鼠标悬停在 **firstMember** 变量上并展开数据提示以检查它。 
  * 注意其 **KindText** 是 **NamespaceDeclaration**。
  * 注意其运行时类型是 **NamespaceDeclarationSyntax**。 

10) 将此节点转换为 **NamespaceDeclarationSyntax** 并存储在新变量中：
```C#
            var helloWorldDeclaration = (NamespaceDeclarationSyntax)firstMember;
```

11) 执行此语句并检查 **helloWorldDeclaration** 变量。
  * 注意与 **CompilationUnitSyntax** 一样，**NamespaceDeclarationSyntax** 也有 **Members** 集合。

12) 检查 **Members** 集合。
  * 注意它包含一个成员。检查它。
    * 注意其 **KindText** 是 **ClassDeclaration**。
    * 注意其运行时类型是 **ClassDeclarationSyntax**。

13) 将此节点转换为 **ClassDeclarationSyntax** 并存储在新变量中：
```C#
            var programDeclaration = (ClassDeclarationSyntax)helloWorldDeclaration.Members[0];
```

14) 执行此语句。

15) 在 **programDeclaration.Members** 集合中找到 **Main** 声明并存储在新变量中：
```C#
            var mainDeclaration = (MethodDeclarationSyntax)programDeclaration.Members[0];
```

16) 执行此语句并检查 **MethodDeclarationSyntax** 对象的成员。
  * 注意 **ReturnType** 和 **Identifier** 属性。
  * 注意 **Body** 属性。
  * 注意 **ParameterList** 属性；检查它。
    * 注意它包含参数列表的左括号和右括号，以及参数列表本身。
    * 注意参数以 **SeparatedSyntaxList**<**ParameterSyntax**> 的形式存储。

17) 将 **Main** 声明的第一个参数存储在变量中。 
```C#
            var argsParameter = mainDeclaration.ParameterList.Parameters[0];
```

18) 执行此语句并检查 **argsParameter** 变量。
  * 检查 **Identifier** 属性；注意它是结构类型 **SyntaxToken**。
  * 检查 **Identifier** **SyntaxToken** 的属性；注意标识符的文本可以在 **ValueText** 属性中找到。

19) 停止程序。
  * 在 Visual Studio 中，选择 **Debug -> Stop Debugging**。

20) 你的程序现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
 
namespace GettingStartedCS
{
    class Program
    {
        static void Main(string[] args)
        {
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
 
            var firstMember = root.Members[0];
 
            var helloWorldDeclaration = (NamespaceDeclarationSyntax)firstMember;
 
            var programDeclaration = (ClassDeclarationSyntax)helloWorldDeclaration.Members[0];
 
            var mainDeclaration = (MethodDeclarationSyntax)programDeclaration.Members[0];
 
            var argsParameter = mainDeclaration.ParameterList.Parameters[0];
 
        }
    }
}
```

### 查询方法
除了使用 **SyntaxNode** 派生类的属性遍历树之外，还可以使用 **SyntaxNode** 上定义的查询方法探索语法树。熟悉 XPath 的人应该对这些方法非常熟悉。你可以将这些方法与 LINQ 结合使用，快速在树中查找内容。 

#### 示例 - 使用查询方法
1) 使用 IntelliSense，通过 root 变量检查 **SyntaxNode** 类的成员。
  * 注意查询方法，如 **DescendantNodes**、**AncestorsAndSelf** 和 **ChildNodes**。

2) 将以下语句添加到 Main 方法末尾。第一条语句使用 LINQ 表达式和 **DescendantNodes** 方法定位与上一示例中相同的参数：
```C#
var firstParameters = from methodDeclaration in root.DescendantNodes()
                                                    .OfType<MethodDeclarationSyntax>()
                      where methodDeclaration.Identifier.ValueText == "Main"
                      select methodDeclaration.ParameterList.Parameters.First();
 
var argsParameter2 = firstParameters.Single();
```

3) 开始调试程序。

4) 打开**即时窗口**。
  * 在 Visual Studio 中，选择 **Debug -> Windows -> Immediate**。

5) 使用即时窗口，输入表达式 **argsParameter == argsParameter2** 并按 Enter 求值。 
  * 注意 LINQ 表达式找到了与手动导航树时相同的参数。

6) 停止程序。

### SyntaxWalker
你通常会希望在语法树中找到特定类型的所有节点，例如文件中的每个属性声明。通过继承 **CSharpSyntaxWalker** 类并重写 **VisitPropertyDeclaration** 方法，你可以在不事先了解语法树结构的情况下处理语法树中的每个属性声明。**CSharpSyntaxWalker** 是 **SyntaxVisitor** 的一种特定类型，它递归地访问节点及其每个子节点。

#### 示例 - 实现 SyntaxWalker
此示例展示如何实现一个 **CSharpSyntaxWalker**，它检查整个语法树并收集它找到的任何未导入 **System** 命名空间的 **using** 指令。

1) 创建一个新的 C# **Stand-Alone Code Analysis Tool** 项目；将其命名为"**UsingCollectorCS**"。

2) 将以下 using 指令添加到 **Program.cs** 文件：
```C#
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
```

3) 将以下代码输入到 **Main** 方法中：
```C#
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@"using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
 
namespace TopLevel
{
    using Microsoft;
    using System.ComponentModel;
 
    namespace Child1
    {
        using Microsoft.Win32;
        using System.Runtime.InteropServices;
 
        class Foo { }
    }
 
    namespace Child2
    {
        using System.CodeDom;
        using Microsoft.CSharp;
 
        class Bar { }
    }
}");
 
            var root = (CompilationUnitSyntax)tree.GetRoot();
```

4) 注意此源文本包含分散在四个不同位置的 **using** 指令：文件级、顶级命名空间以及两个嵌套命名空间中。

5) 向项目添加新类文件。
  * 在 Visual Studio 中，选择 **Project -> Add New Item...** 
  * 在"添加新项"对话框中输入 **UsingCollector.cs** 作为文件名。

6) 将以下 using 指令添加到 UsingCollector.cs 文件顶部 
```C#
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
```

7) 使此文件中的新 **UsingCollector** 类继承 **CSharpSyntaxWalker** 类：
```C#
    class UsingCollector : CSharpSyntaxWalker
```

8) 在 **UsingCollector** 类中声明一个公共只读字段；我们将使用此变量存储找到的 **UsingDirectiveSyntax** 节点：
```C#
        public readonly List<UsingDirectiveSyntax> Usings = new List<UsingDirectiveSyntax>();
```

9) 重写 **VisitUsingDirective** 方法：
```C#
        public override void VisitUsingDirective(UsingDirectiveSyntax node)
        {
            
        }
```

10) 使用 IntelliSense，通过此方法的 **node** 参数检查 **UsingDirectiveSyntax** 类。
  * 注意 **NameSyntax** 类型的 **Name** 属性；它存储被导入命名空间的名称。

11) 将 **VisitUsingDirective** 方法中的代码替换为以下内容，如果 **Name** 不引用 **System** 命名空间或其任何子命名空间，则有条件地将找到的 **node** 添加到 **Usings** 集合中：
```C#
            if (node.Name.ToString() != "System" &&
                !node.Name.ToString().StartsWith("System."))
            {
                this.Usings.Add(node);
            }
```

12) **UsingCollector.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
 
namespace UsingCollectorCS
{
    class UsingCollector : CSharpSyntaxWalker
    {
        public readonly List<UsingDirectiveSyntax> Usings = new List<UsingDirectiveSyntax>();

        public override void VisitUsingDirective(UsingDirectiveSyntax node)
        {
            if (node.Name.ToString() != "System" &&
                !node.Name.ToString().StartsWith("System."))
            {
                this.Usings.Add(node);
            }
        }
    }
}
```

13) 返回 **Program.cs** 文件。

14) 将以下代码添加到 **Main** 方法末尾，以创建 **UsingCollector** 的实例，使用该实例访问解析树的根，并遍历收集到的 **UsingDirectiveSyntax** 节点，将其名称打印到 **Console**：
```C#
            var collector = new UsingCollector();
            collector.Visit(root);
 
            foreach (var directive in collector.Usings)
            {
                Console.WriteLine(directive.Name);
            }
```

15) 你的 **Program.cs** 文件现在应如下所示：
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
 
namespace UsingCollectorCS
{
    class Program
    {
        static void Main(string[] args)
        {
            SyntaxTree tree = CSharpSyntaxTree.ParseText(
@"using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
 
namespace TopLevel
{
    using Microsoft;
    using System.ComponentModel;
 
    namespace Child1
    {
        using Microsoft.Win32;
        using System.Runtime.InteropServices;
 
        class Foo { }
    }
 
    namespace Child2
    {
        using System.CodeDom;
        using Microsoft.CSharp;
 
        class Bar { }
    }
}");
 
            var root = (CompilationUnitSyntax)tree.GetRoot();
 
            var collector = new UsingCollector();
            collector.Visit(root);
 
            foreach (var directive in collector.Usings)
            {
                Console.WriteLine(directive.Name);
            }
        }
    }
}
```

16) 按 **Ctrl+F5** 运行程序（不调试）。你应该会看到以下输出：

```
Microsoft.CodeAnalysis
Microsoft.CodeAnalysis.CSharp
Microsoft
Microsoft.Win32
Microsoft.CSharp
Press any key to continue . . .
```

17) 观察到 walker 已在所有四个位置找到了所有非 **System** 命名空间的 **using** 指令。

18) 恭喜！你刚刚使用了 **Syntax API** 在 C# 源代码中定位特定类型的 C# 语句和声明。
