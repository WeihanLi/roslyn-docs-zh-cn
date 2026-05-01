# .NET 8 上编辑并继续（EnC）和热重载支持的编辑操作
热重载允许你在应用程序运行时修改/添加源代码，无论调试器是否附加到进程（F5）或未附加（Ctrl+F5）。
编辑并继续允许你在调试中断模式下修改/添加源代码，而无需重启调试会话。

本文档记录了当前状态。该领域潜在的未来改进正在 https://github.com/dotnet/roslyn/issues/49001 中跟踪。

**定义**
* [变量捕获](http://blogs.msdn.com/b/matt/archive/2008/03/01/understanding-variable-capturing-in-c.aspx) 是一种机制，通过该机制，内联定义的 lambda/委托能够持有其词法作用域内的任何变量
* **作用域** 是程序文本中可以不加限定地通过名称引用该名称所声明的实体的区域
* **调试语句** 是由相邻序列点分隔的指令范围。通常调试语句对应于一条语言语句，但也可能只对应语言语句的一部分（例如块语句的左花括号）、一个表达式（例如 lambda 主体）或其他连续语法（基构造函数调用）。
* **内部活动语句** 是包含堆栈帧返回地址的调试语句。
* **叶活动调试语句** 是包含任意线程的 IP（指令指针）的调试语句。


### 支持的编辑操作
| 编辑操作 | 附加信息 |
| ------------------- |--------------------|
| 向现有类型添加方法、字段、构造函数、属性、事件、索引器、字段和属性初始化器、嵌套类型及顶级类型（包括委托、枚举、接口、抽象和泛型类型以及匿名类型）  | 现有类型不能是接口。<br/> <br/> 不支持在现有枚举中添加或修改[枚举成员](https://msdn.microsoft.com/en-us/library/sbbt4032.aspx)。 |
| 添加和修改迭代器、`yield` 语句 | 支持将普通方法更改为迭代器方法 |
| 添加和修改异步方法、`await` 表达式 | 不支持修改包装在其他表达式内的 await 表达式（例如 ```G(await F());```）。<br/><br/> 支持将普通方法更改为异步方法。 |
| 添加和修改对动态对象的操作 | - |
| 添加 lambda 表达式 | 只有在以下情况下才能添加 lambda 表达式：lambda 是静态的、访问已被捕获的"this"引用，或访问来自单个作用域的已捕获变量 |
| 修改泛型代码 | 在 .NET 8 和 Visual Studio 17.7 中启用 |
| 修改 lambda 表达式 | 以下规则确保生成的闭包树结构不会改变——从而确保新主体中的 lambda 映射到实现其先前版本的相应 CLR 方法：<ul><li>不能修改 lambda 签名（包括参数的名称、类型、引用性以及返回类型）</li><li>lambda 表达式捕获的变量集不能修改（修改前未被捕获的变量不能在修改后被捕获，反之亦然）</li><li>不能修改捕获变量的作用域</li><li>不能修改 lambda 表达式访问的已捕获变量集</li></ul> |
| 添加 LINQ 表达式 | LINQ 表达式包含隐式声明的匿名函数。这意味着 lambda 和 LINQ 的编辑规则相同。 |
| 修改 LINQ 表达式 | LINQ 表达式包含隐式声明的匿名函数。这意味着 lambda 和 LINQ 的编辑规则相同。 |
| 组合修改异步 lambda 和 LINQ 表达式 | 你可以编辑各种嵌套表达式，前提是它们满足 EnC 规则 | 
| 编辑分部类 | 在 [VS 16.10](https://learn.microsoft.com/en-us/visualstudio/releases/2019/release-notes-v16.10) 中启用 |
| 编辑源生成文件 | 在 [VS 16.10](https://learn.microsoft.com/en-us/visualstudio/releases/2019/release-notes-v16.10) 中启用。 |
| 添加 using 指令 | 在 [VS 16.10](https://learn.microsoft.com/en-us/visualstudio/releases/2019/release-notes-v16.10) 中启用。 |
| 删除字段以外的成员 | 在 [VS 17.3](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.3) 中启用 |
| 重命名字段以外的成员 | 在 [VS 17.4](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.4) 中启用 |
| 重命名方法参数 | 在 [VS 17.0](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.0) 中启用 | 
| 修改方法参数类型 | 在 [VS 17.4](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.4) 中启用 |
| 更改方法/事件/属性/运算符/索引器的返回类型 | 在 [VS 17.4](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.4) 中启用 |
| 添加和修改自定义属性 | 在 [VS 17.0](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.0) 中启用 | 
| 添加和修改命名空间声明 | 在 [VS 17.3](https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes-v17.3) 中启用 |

### 不支持的编辑操作
| 编辑操作 | 附加信息 |
| ------------------- |-----------------|
| 修改接口 | - |
| 添加方法体使抽象方法变为非抽象方法 | - |
| 向类型添加新的抽象、虚拟或重写成员 | 你**可以**向抽象类型添加非抽象成员 |
| 向现有类型添加[析构函数](https://msdn.microsoft.com/en-us/library/66x5fx1b.aspx) |  - |
| 修改类型参数、基类型、委托类型 | - |
| 修改包含活动语句（叶或内部）的 catch 块 | - |
| 修改 try-catch-finally 块（如果 finally 子句包含活动语句） | - |
| 删除类型 | - |
| 编辑引用嵌入式互操作类型的成员 | - |
| 编辑包含 On Error 或 Resume 语句的成员 | 特定于 Visual Basic |
| 编辑包含 Aggregate、Group By、Simple Join 或 Group Join LINQ 查询子句的成员 | 特定于 Visual Basic |
| 在包含项目（这些项目在 `AssemblyVersionAttribute` 中指定了 `*`，例如 `[assembly: AssemblyVersion("1.0.*")`]）的解决方案中进行编辑。 | 参见下方[解决方法](#projects-with-variable-assembly-versions)。 |

### 使用可变程序集版本的项目

热重载和编辑并继续与在 `AssemblyVersionAttribute` 值中使用 `*` 不兼容。版本中存在 `*` 意味着编译器会
根据当前时间在每次构建时生成新版本。这样的构建会产生不可重现、不确定性的输出（参见[可重现构建](https://reproducible-builds.org)）。

因此强烈建议使用其他版本控制方法，例如 [Nerdbank.GitVersioning](https://github.com/dotnet/Nerdbank.GitVersioning)，它从 HEAD 提交 SHA（对于 git 存储库）派生程序集版本。

> 要使用 `Nerdbank.GitVersioning` 包启用基于提交 SHA 的程序集版本生成，请在 `version.json` 中指定 `{ "assemblyVersion" : {"precision": "revision"} }` 设置。

如果你希望继续在 `AssemblyVersionAttribute` 中使用 `*`，建议使用条件编译指令，仅将此类版本应用于发布构建，如下所示：

```C#
#if DEBUG
[assembly: AssemblyVersion("1.0.0.0")]
#else
[assembly: AssemblyVersion("1.0.*")]
#endif
```
