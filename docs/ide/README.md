# 最近添加的生产力功能和发布说明链接

## 目录
 * [Dev16](#prod-16)
   * [Dev 16.9](#prod-16-9)
   * [Dev 16.8](#prod-16-8) 
   * [Dev 16.7](#prod-16-7)
   * [Dev 16.6](#prod-16-6)
   * [Dev 16.5](#prod-16-5)
   * [Dev 16.4](#prod-16-4)
   * [Dev 16.3](#prod-16-3)
   * [Dev 16.2](#prod-16-2)
   * [Dev 16.1](#prod-16-1)
   * [Dev16.0](#prod-16-0)
 * [Dev15](#prod-15)
   * [Dev 15.9](#prod-15-9)
   * [Dev 15.8](#prod-15-8)
   * [Dev 15.7](#prod-15-7)
   * [Dev 15.6](#prod-15-6)
   * [Dev 15.5](#prod-15-5)
   * [Dev 15.4](#prod-15-4)

## <a id="prod-16"></a> dev16

### <a id="prod-16-7"></a> 16.9 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.9))
* 现在有预处理器符号的 IntelliSense 补全。
* 解决方案资源管理器现在显示新的 .NET 5.0 源生成器。
* Go To All 不会在 netcoreapp3.1 和 netcoreapp2.0 之间显示重复结果。
* Quick Info 现在为禁止显示编译器警告 ID 或编号。
* 将类型复制粘贴到新文件时，Using 指令现在会自动添加。
* 按 `;` 从补全列表中接受方法时，IntelliSense 现在会自动插入括号和分号用于对象创建和方法调用。
* C# 9.0 [记录](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#record-types&preserve-view=true) 的语义着色。
* 删除不必要的丢弃的重构。
* 将逐字字符串和常规字符串转换为插值字符串的重构，保留打算输出的花括号。
* Visual Basic 中的代码修复，当将共享方法转换为模块时删除 shared 关键字。
* 在非争议场景中建议使用 `new(…)` 的重构
* 为 C# 和 Visual Basic 删除冗余相等表达式的代码修复
* .NET 代码样式 (IDE) 分析器现在可以在构建时强制执行
* 语法可视化工具显示增强颜色的当前前景色
* 悬停在 pragma 警告的诊断 ID 上时的新工具提示
* 从注释内键入回车键时，新行现在会自动注释掉
* 内联参数名称提示增强
* 使用 WSL 2 进行 .NET Core 调试

### <a id="prod-16-7"></a> 16.8 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.8))
* Roslyn 分析器现在包含在 .NET 5.0 SDK 中
* 当存在禁止运算符时引入新的 C# 9 `not` 模式匹配语法的重构
* 内联方法重构，帮助替换单语句体内的静态、实例和扩展方法的用法
* 将 C# 中的 `typeof` 实例转换为 `nameof`，将 Visual Basic 中的 `GetType` 转换为 `NameOf` 的代码修复
* C# 和 Visual Basic 支持内联参数名称提示，在函数调用中的每个参数之前为文字、强制转换文字和对象实例化插入装饰
* 从选定类中提取成员到新基类的重构，适用于 C# 和 Visual Basic
* 代码清理有新的配置选项，可以在单个文件或整个解决方案中应用 EditorConfig 文件中设置的格式和文件头首选项
* 删除 `in` 关键字的代码修复，其中参数不应按引用传递
* 引入新的 C#9 模式组合器和模式匹配建议的重构，如在适用的情况下将 `==` 转换为使用 `is`
* 当尝试在非抽象类中编写抽象方法时使类抽象的代码修复
* `DateTime` 和 `TimeSpan` 字符串文字中的 IntelliSense 补全在键入第一个引号时自动出现
* 删除不必要的 `pragma suppressions` 和不必要的 `SuppressMessageAttributes` 的代码修复
* `Rename` 和 `Find All` 引用理解全局 `SuppressMessageAttributes` 目标字符串内的符号引用
* _ByVal_ 淡出表示它不是必需的，以及在 Visual Basic 中删除不必要的 _ByVal_ 的代码修复
* 多运行时的交互式窗口支持，如 .NET Framework 和 .NET Core。
* 添加了新的 _RegisterAdditionalFileAction_ API，允许分析器作者为附加文件创建分析器。

### <a id="prod-16-7"></a> 16.7 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.7))
* 当禁止运算符存在但没有效果时，现在有警告和代码修复
* 还提供建议正确否定表达式的第二个代码修复
* 现在可以在类型中生成构造函数时生成属性
* Quick Info 现在显示诊断 ID 以及帮助链接
* 现在有一个快速操作可以向类添加调试器显示属性
* 现在有意外赋值或与同一变量比较的代码修复
* 现在可以为实现 IComparable 的类型生成比较运算符
* 现在可以在为结构生成 .Equals 时生成 IEquatable 运算符
* 现在可以为所有未使用的构造函数参数创建和分配属性或字段
* 现在在 DateTime 和 TimeSpan 字符串文字中有 IntelliSense 补全
* 现在可以在更改签名对话框中添加参数
* 分析器作者现在可以在使用 NuGet 发布分析器时使用 CompletionProviders 进行 IntelliSense 补全

### <a id="prod-16-6"></a> 16.6 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.6))
* 添加显式转换代码修复
* 简化条件表达式重构
* 将常规字符串文字转换为逐字字符串文字（反之亦然）重构
* 通过编辑器批量配置分析器类别的严重性级别
* Quick Info 样式支持包含 returns 和 value 标签的 XML 注释
* 使用 EditorConfig 向现有文件、项目和解决方案添加文件头
* [Microsoft Fakes 对 .NET Core 的支持](https://docs.microsoft.com/visualstudio/releases/2019/release-notes#microsoft-fakes-for-net-core-and-sdk-style-projects) 和 SDK 样式项目

### <a id="prod-16-5"></a> 16.5 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.5))
* 当 System.HashCode 可用时，现在可以使用 System.HashCode 实现 GetHashCode 方法。
* 将 if 语句转换为 switch 语句或 switch 表达式的重构。
* 未导入扩展方法的 IntelliSense 补全。您首先需要在 **工具 > 选项 > 文本编辑器 > C# > Intellisense > 并选择显示未导入命名空间中的项** 中打开此选项。

### <a id="prod-16-4"></a> 16.4 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.4))
* 现在可以直接通过编辑器（通过 **Ctrl+.** 菜单）或错误列表（右键单击错误、警告或消息）配置代码样式规则的严重性级别。这也将更新 EditorConfig 并适用于第三方分析器。
* Find All References 现在允许您按类型和成员分组。
* 使本地函数静态化的重构
* 在本地静态函数中显式传递变量的代码修复。
* Go To Base 命令可以向上导航继承链。可在上下文（右键单击）菜单中使用，或者您可以键入（Alt+Home）。
* 为所有参数添加空检查的重构。
* 没有 XML 文档的方法现在可以自动从它重写的方法继承 XML 文档。将光标放在实现已记录接口方法的未记录方法上。然后 Quick Info 将显示来自接口方法的 XML 文档。

### <a id="prod-16-3"></a> 16.3 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.3))
* 使用重构包装流式调用链
* 在写入初始化器后立即引入局部变量
* 代码分析页面现在在项目属性中。右键单击解决方案资源管理器中的项目名称并选择属性。选择代码分析以安装分析器包并配置何时运行代码分析。
* 现在，对于关闭未导入类型补全的用户，通过添加到 IntelliSense 切换的新导入类型过滤器，可以更轻松地将其重新添加到补全列表中。
* 现在有 XML 注释的 Quick Info 样式支持。将光标放在方法名称上。然后 Quick Info 将显示代码上方 XML 注释中支持的样式。
* 重命名接口、枚举或类时重命名文件。将光标放在类名中并键入（Ctrl + R,R）以打开重命名对话框并选中"重命名文件"框。
* 现在有多目标项目的 Edit and Continue 支持，包括在不同域或加载上下文中在同一进程中多次加载的模块。此外，即使包含项目未加载或应用程序正在运行，开发人员也可以编辑源文件。

### <a id="prod-16-2"></a> 16.2 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.2))
* 将 Sort Usings 作为与 Remove Usings 分开的命令恢复。可在 Edit > IntelliSense 下使用。
* 将 switch 语句转换为 [switch 表达式](https://docs.microsoft.com/dotnet/csharp/whats-new/csharp-8#switch-expressions)（验证您正在使用 C# 8 以获取 switch 表达式功能）
* 生成参数
* [测试资源管理器改进](https://docs.microsoft.com/visualstudio/releases/2019/release-notes#test-explorer)
  * 显著减少 Visual Studio 进程消耗的内存，并加快具有大量测试的解决方案的测试发现
  * 通过、失败、未运行测试的过滤按钮
  * "运行失败测试"和"运行上次测试运行"的附加按钮
  * 自定义测试资源管理器中显示的列
  * 指定测试层次结构每一层中显示的内容。默认层是项目、命名空间，然后是类，但附加选项包括结果或持续时间分组。
  * 测试状态窗口（显示消息、输出等的测试列表下方的窗格）更加可用。用户可以复制文本子字符串，并且字体宽度固定以获得更可读的输出。
  * 播放列表可以显示在多个选项卡中，并且更容易根据需要创建和丢弃。
  * Live Unit Testing 现在在测试资源管理器中有自己的视图。它显示当前包含在 Live Unit Testing 中的所有测试（即实时测试集），因此测试人员可以轻松地将 Live Unit Testing 结果与手动运行的测试结果分开跟踪。
  * 有一个目标框架列可以显示多目标测试结果。

### <a id="prod-16-1"></a> 16.1 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.1))
* 现在有未导入类型的实验性 intellisense 补全！现在您可以获得项目依赖项中类型的 intellisense 建议，即使您尚未将导入语句添加到文件中。您必须在 Tools > Options > Text Editor > C# > Intellisense 中打开此选项。
* 切换单行注释/取消注释现在可通过键盘快捷键（Ctrl+K,/）使用。此命令将根据您的选择是否已注释来添加或删除单行注释。
* 现在可以使用位于 Tools > Options > Text Editor > C# > Code Style 中的"生成 editorconfig"按钮导出命名样式。
* 现在可以使用新的 editorconfig 代码样式规则来要求或阻止命名空间内的 usings。当您使用位于 Tools > Options > Text Editor > C# > Code Style 中的"生成 editorconfig"按钮时，此设置也会被导出。
* Find All References "Kind" 列现在有更多过滤选项，并且可以识别命名空间和类型。
* .NET 代码修复和重构
  * 拆分/合并 if
  * 包装二元表达式
  * 解封类
* 现在可以通过 intellisense 菜单（Ctrl + space）在正则表达式字符串内访问正则表达式补全列表。这些补全还包括对建议作用的内联描述。
* 现在可以对项目和解决方案使用一键代码清理。您可以在解决方案资源管理器中右键单击项目或解决方案并选择"运行代码清理"。
* 现在可以使用重构对话框将类型移动到命名空间或文件夹。将光标放在类名中，键入（Ctrl + .）以打开快速操作和重构菜单，然后选择"移动到命名空间"。这将启动一个对话框，您可以在其中选择要将类型移动到的目标命名空间。
* 切换块注释/取消注释（Ctrl+Shift+/）或通过 Edit > Advanced > Toggle Block Comment。
* 现在有一个使只读结构字段可写的代码修复。将光标放在结构名称中，键入（Ctrl+.）以打开快速操作和重构菜单，然后选择"使只读字段可写"。
* 从构造函数添加私有字段的代码修复（反之亦然）更容易发现，并且当选择字段名称的任何部分时都会显示。此重构现在还提供所有可能的构造函数。

### <a id="prod-16-0"></a> 16.0 ([发布说明](https://docs.microsoft.com/visualstudio/releases/2019/release-notes-v16.0))

* .NET 重构和代码修复：
  * 同步命名空间和文件夹名称
  * 使用对话框选项将成员提升到基类的重构
  * 包装/缩进/对齐参数/参数列表
  * 将匿名类型转换为元组
  * 为 lambda 使用表达式/块体
  * 反转条件表达式和逻辑操作
  * 转换为复合赋值
  * 在 "/" 上自动关闭块注释
  * 转换为复合赋值
  * 修复隐式类型变量不能是常量
  * 在键入插值逐字字符串时自动修复将 @$" 替换为 $@"
  * #nullable enable|disable 的补全
  * 修复未使用的表达式值和参数
  * 修复允许 Extract Interface 保留在同一文件中
   * 对于 "await" 被暗示但省略的情况，现在有编译器警告。
   * 将本地函数转换为方法。
   * 将元组转换为命名结构。
   * 将匿名类型转换为类。
   * 将匿名类型转换为元组。
   * 将 foreach 循环转换为 LINQ 查询或 LINQ 方法。
   * 生成解构方法
* 在 Find All References 窗口中按读/写对引用进行分类。
* 为 csharp_prefer_braces 添加 Editorconfig when_multiline 选项。
* 新的分类颜色可从 .NET 编译器平台 SDK（又名 Roslyn）获得。类似于 Visual Studio Code 颜色的新默认颜色正在逐步推出。您可以在 Tools > Options > Environment > Fonts and Colors 中调整这些颜色，或者通过取消选中 Use enhanced colors 复选框在 Environment > Preview Features 中关闭它们。我们希望听到关于此更改如何影响您的工作流程的反馈。
* 配置[代码清理](https://docs.microsoft.com/visualstudio/ide/code-styles-and-code-cleanup)（以前是 Format Document 的一部分的设置）
* [Format document 全局工具](https://docs.microsoft.com/visualstudio/releases/2019/release-notes#-apply-code-style-preferences)可以从命令行运行
 * 使用[代码度量](https://docs.microsoft.com/visualstudio/code-quality/code-metrics-values)与 .NET Core 项目，通过我们添加的兼容性。
 * 通过 **Tools > Options > Text Editor > C# > Code Style** 使用"从设置生成 .editorconfig 文件"按钮将编辑器设置导出到 Editorconfig 文件。
 * 使用 C# 和 Visual Basic 的新 Regex 解析器支持。正则表达式现在被识别，并在其上启用语言功能。当字符串传递给 Regex 构造函数或字符串前面紧跟包含字符串 language=regex 的注释时，正则表达式字符串被识别。此版本中包含的语言功能是分类、括号匹配、突出显示引用和诊断。
 * 现在可以使用死代码分析来查找未使用的私有成员，并提供可选的代码修复来删除未使用的成员声明。
 * 访问器上的 Find References 功能现在只返回该访问器的结果。
 * 将代码粘贴到文件中时可以添加 "Using" 语句。粘贴识别的代码后会出现代码修复，提示您添加相关的缺失导入。
 * 现在可以使用 Find All References (Shift-F12) 和 CodeLens 显示 .NET Core 项目中 Razor (.cshtml) 文件的结果。然后您可以导航到相关 Razor 文件中的标识代码。
 * 使用 FxCop 运行代码分析时，您现在将收到警告。.NET 编译器分析器是未来执行代码分析的推荐方式。阅读更多关于迁移到 .NET 编译器平台分析器的信息。

## <a id="prod-15"></a> dev15

### dev15 摘要
* 通过 Format Document 进行基于 Visual Studio 的代码清理
* 重构：InvertIf，从方法调用点添加参数，删除不必要的括号，将 if-else 赋值、属性补全和 return 转换为三元条件。
* .editorconfig "刷新"，以便更改将自动应用，无需文件重新加载
* IOperation API RTW
* 便携式 PDB 格式以启用跨平台调试场景

### <a id="prod-15-9"></a>15.9
无

### <a id="prod-15-8"></a>15.8 ([发布说明](https://docs.microsoft.com/visualstudio/releasenotes/vs2017-relnotes-v15.8#productivity))
 * C# 开发的 Format Document (Ctrl + K, D 或 Ctrl + E, D)。
 * .NET 重构和代码修复：
   * 反转 If
   * 从方法调用点添加参数
   * 删除不必要的括号
   * 在赋值和 return 语句中使用三元条件
 * Go to Enclosing Block (Ctrl + Alt + UpArrow)
 * Go to Next/Previous Issue (Alt + PgUp/PgDn) 跳到错误、波浪线、灯泡。
 * Go to Member (Ctrl + T, M) 现在默认范围为文件
 * 多光标：使用 Ctrl + Alt + LeftMouseClick 插入光标。
 * 多光标：使用 Shift + Alt + Ins 在与当前选择匹配的下一个位置添加选择和光标。
 * 使用 Alt + ` 访问上下文导航菜单。
 * Visual Studio Code 和 ReSharper (Visual Studio) 键绑定

### <a id="prod-15-7"></a>15.7 ([发布说明](https://docs.microsoft.com/visualstudio/releasenotes/vs2017-relnotes-v15.7#dotnet_productivity))
* .NET 重构和代码修复：
   * For 到 foreach，反之亦然。
   * 使私有字段只读。
   * 在 var 和显式类型之间切换，无论您的代码样式首选项如何。
 * Go To Definition (F12) 现在支持 LINQ 查询子句和解构。
 * Quick Info 显示 lambda 和本地函数上的捕获，因此您可以看到哪些变量在范围内。
 * Change Signature 重构 (Ctrl+. on signature) 适用于本地函数。
 * 您可以就地编辑 .NET Core 项目文件，因此完全支持打开包含文件夹、恢复选项卡和其他编辑器功能。IDE 更改，如添加链接文件，将与编辑器中未保存的更改合并。

### <a id="prod-15-6"></a>15.6 ([发布说明](https://docs.microsoft.com/visualstudio/releasenotes/vs2017-relnotes-v15.6#productivity))
 * 导航到反编译源（必须在 Tools > Options > Text Editor > C# > Advanced > Enable navigation to decompiled sources 中启用）
 * .NET EditorConfig 选项：dotnet_prefer_inferred_tuple_names
 * .NET EditorConfig 选项：dotnet_prefer_inferred_anonymous_type_member_names
 * Ctrl + D 复制文本
 * 展开选择命令允许您依次将选择扩展到下一个逻辑块。您可以使用快捷键 Shift+Alt+= 展开和 Shift+Alt+- 收缩当前选择。

### <a id="prod-15-5"></a>15.5
无

### <a id="prod-15-4"></a>15.4 ([发布说明](https://docs.microsoft.com/visualstudio/releasenotes/vs2017-relnotes-v15.4#editor))
 * 编辑器：Control Click Go To Definition
