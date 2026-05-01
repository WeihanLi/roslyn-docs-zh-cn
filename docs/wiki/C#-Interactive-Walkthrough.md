本演练是学习基本交互式概念以及如何使用 C# 交互式窗口的入门指南。要深入了解交互式窗口，请观看[此视频](https://channel9.msdn.com/Events/Visual-Studio/Connect-event-2015/103)或查阅[我们的文档](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Interactive-Window.md)。

*注意*：本演练改编自 [Bill Chiles](https://github.com/billchi-ms) 的原版内容。感谢 Bill！

## 简介：什么是交互式？

借助全新的 **C# 交互式窗口**，你可以立即看到表达式的返回值或 API 调用的执行结果。交互式窗口与即时窗口类似，但交互式窗口有许多改进，例如 IntelliSense 功能以及重新定义函数和类的能力。在 REPL 提示符处输入代码片段后，代码会立即执行。你可以输入语句和表达式，以及类和函数定义。无需创建项目、定义类、定义 Main、将表达式写在 ```Console.WriteLine()``` 调用中，也不需要为了保持 cmd.exe 窗口活跃而编写虚假的 ```Console.ReadLine```。当你输入每一行完整的表达式或构造并按下 ```Enter``` 时，代码即会执行。如果按下 ```Enter``` 时输入的代码尚未完整，则不会执行，你可以继续输入代码。

交互式窗口是一个"读取-求值-打印"循环（即 [REPL](http://en.wikipedia.org/wiki/REPL)）。如你所见，REPL 是能提升生产力的工具，长期以来一直是动态语言和函数式语言的专属领域。Roslyn 为 C# 提供的 REPL 将你所熟悉的 VS 优点——如代码补全、语法着色、快速修复等——带入了 REPL 体验。

要使用本演练，你必须先安装 [Visual Studio 2015 Update 1](http://go.microsoft.com/fwlink/?LinkId=691129)。

## 演练

本演练演示了 C# 交互式窗口的以下特性或属性：
- 以默认执行上下文启动 REPL
- 使用指令
- 使用 ```Alt+UpArrow``` 和 ```Alt+DownArrow``` 进行历史命令导航
- 当输入完整时按 Enter 执行
- ```Ctrl+Enter``` 强制执行当前输入，或在光标位于之前提交内容时获取之前的输入
- ```Shift+Enter``` 插入换行而不执行当前输入
- Var 声明
- 着色、补全、参数提示
- 表达式求值
- 多行输入
- 带编辑功能的多行历史
- 演示将控制台 I/O 重定向到 REPL

### 步骤
1. 打开 [Visual Studio 2015 Update 1](http://go.microsoft.com/fwlink/?LinkId=691129)

2. 在**视图**菜单中，选择**其他窗口**，然后选择 **C# 交互式**。最好将其拖到文档区域并停靠在文档区域底部。通常的做法是在编辑器缓冲区和 REPL 之间切换，用于输入和保存代码片段，尤其是在编写脚本时。

3. 为了向我们的传统致敬，让我们在 REPL 中输入必不可少的"Hello World"程序。输入以下内容，然后按 ```Enter```：
  
```csharp
> Console.Write("Hello, World!")
```
  
4. REPL 中有一些看起来像指令的命令。```#help``` 命令描述了常用命令（以 # 开头的输入）及入门快捷方式：

```csharp
> #help
```
 
5. 输入以下内容，然后按 ```Enter```。注意 IntelliSense 补全在你输入时的帮助。
 
```csharp
> using System.IO;
```

然后输入以下命令，你可以通过 ```Alt+UpArrow``` 调用历史记录，用退格键删除"IO;"，然后输入"Net;"并按 ```Enter``` 来完成。

```csharp
> using System.Net;
```

6. 输入以下不带末尾分号的不完整语句，然后按 ```Enter```

```csharp
> var url =
```

由于语句尚未完整，REPL 不会执行它，代码会继续到下一行。粘贴以下代码，然后按 ```Enter```。此代码是上一行的延续。
  
```csharp
"https://download.microsoft.com/download/4/C/8/4C830C0C-101F-4BF2-8FCB-32D9A8BA906A/Import_User_Sample_en.csv";
```
  
由于该行已完整，在输入末尾按 ```Enter``` 即可执行代码。
  
7. 现在通过将变量作为表达式输入来查看其内容。只输入变量名，末尾不加分号：

```csharp
> url
```

8. 这是查看表达式求值结果的另一种方式。输入以下内容并按 ```Enter```。

```csharp
> Console.Write("url: " + url)
```
  
9. 在交互式窗口中输入以下几行代码。该代码使用 WebRequest 实例从网站下载数据，并将结果存入 ```csv``` 字符串变量。有关更多信息及类似示例，请参阅 [WebRequest 类](https://msdn.microsoft.com/en-us/library/system.net.webrequest.aspx)。

```csharp
> var request = WebRequest.Create(url);
> var response = request.GetResponse();
> var dataStream = response.GetResponseStream();
> var reader = new StreamReader(dataStream);
> var csv = await reader.ReadToEndAsync();
> reader.Close();
> dataStream.Close();
> response.Close();
```
 
注意，我们可以在 REPL 中使用异步代码。REPL 会等待异步代码完成求值后再继续。

10. 现在我们已经获取了一些数据，在继续之前先简要查看一下数据。输入以下行，末尾不加分号：

```csharp
> csv.Length
```
  
  REPL 会在下一行显示 ```csv``` 字符串的长度。
  
11. 好的，数据量很大，也许我们可以将其分割。让我们看看有多少行：

```csharp
> csv.Split('\n').Length
```

12. 只有几行，所以让我们看看前几百个字符，了解字符串的结构或每行的大致长度。输入以下代码：

```csharp
> Console.Write(csv.Substring(0,600))
```
  
你将看到类似如下的输出：
  
```
User Name,First Name,Last Name,Display Name,Job Title,Department,Office Number,Office Phone,Mobile Phone,Fax,Address,City,State or Province,ZIP or Postal Code,Country or Region
chris@contoso.com,Chris,Green,Chris Green,IT Manager,Information Technology,123451,123-555-1211,123-555-6641,123-555-9821,1 Microsoft way,Redmond,Wa,98052,United States
ben@contoso.com,Ben,Andrews,Ben Andrews,IT Manager,Information Technology,123452,123-555-1212,123-555-6642,123-555-9822,1 Microsoft way,Redmond,Wa,98052,United States
david@contoso.com,David,Longmuir,David Longmuir,IT Manager,Information Technology,12
```

13. 现在我们可以看出数据的结构了。让我们构建一个查询，从最后一列提取数据（Skip(1) 跳过标题行）。你可以在行尾使用 ```Shift+Enter``` 来避免在输入完所有内容之前执行；只有当表达式看起来完整时，```Enter``` 才会求值：

```csharp
var users = csv.Split('\n').Skip(1)
                .Select(line => line.Split(','))
                .Where(values => values.Length == 15)
                .Select(values => new { 
                    firstName = values[1], 
                    lastName = values[2], 
                    officeNumber = int.Parse(values[6]) 
                });
```

14. 让我们打印查询结果中的一部分用户。你可以在前两行之后使用 ```Shift+Enter``` 来避免立即执行。如果在 'foreach' 循环内使用 ```Enter```，代码要等到你输入最后一个大括号并按 ```Enter``` 后才会执行。

```csharp
foreach (var u in users)
    Console.WriteLine(u)
```

以下是你最终会话的完整内容：

```csharp
using System.IO;
using System.Net;

var url = "https://download.microsoft.com/download/4/C/8/4C830C0C-101F-4BF2-8FCB-32D9A8BA906A/Import_User_Sample_en.csv";
var request = WebRequest.Create(url);
var response = request.GetResponse();
var dataStream = response.GetResponseStream();
var reader = new StreamReader(dataStream);
var csv = await reader.ReadToEndAsync();
reader.Close();
dataStream.Close();
response.Close();
var users = csv.Split('\n').Skip(1)
                .Select(line => line.Split(','))
                .Where(values => values.Length == 15)
                .Select(values => new {
                    firstName = values[1],
                    lastName = values[2],
                    officeNumber = int.Parse(values[6])
                });

foreach (var u in users)
     Console.WriteLine(u);
```

你已完成演练。享受使用 REPL 的乐趣，欢迎提供反馈！如果你有兴趣深入了解 C# 交互式窗口，请观看[此视频](https://channel9.msdn.com/Events/Visual-Studio/Connect-event-2015/103)或查阅[我们的文档](https://github.com/dotnet/roslyn/blob/main/docs/wiki/Interactive-Window.md)。
