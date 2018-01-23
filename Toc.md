# Roslyn Table of Content

- 分析器
  - [分析器操作语义](./analyzers/Analyzer%20Actions%20Semantics.md)
  - [分析器示例](./analyzers/Analyzer%20Samples.md)
  - [修复所有Provider](./analyzers/FixAllProvider.md)
  - [本地化分析器](./analyzers/Localizing%20Analyzers.md)
  - [报告分析器格式](./analyzers/Report%20Analyzer%20Format.md)
  - [使用更多文件](./analyzers/Using%20Additional%20Files.md)

- [编译器](./compilers/README.md)
  - C#
    - [COM互操作](./compilers/CSharp/COM%20Interop.md)
    - [代码生成的不同](./compilers/CSharp/CodeGen%20Differences.md)
    - [命令行](./compilers/CSharp/CommandLine.md)
    - [编译器变化 - VS2015](./compilers/CSharp/Compiler%20Breaking%20Changes%20-%20VS2015.md)
    - [编译器变化 - VS2017](./compilers/CSharp/Compiler%20Breaking%20Changes%20-%20VS2017.md)
    - [编译器变化- POST VS2017](./compilers/CSharp/Compiler%20Breaking%20Changes%20-%20post%20VS2017.md)
    - [定义和使用动态类型](./compilers/CSharp/Definite%20Assignment%20And%20Dynamic.md)
    - [定义和使用分部类型](./compilers/CSharp/Definite%20Assignment%20And%20Partials.md)
    - [明确定义](./compilers/CSharp/Definite%20Assignment.md)
    - [访问索引属性](./compilers/CSharp/Indexed%20Property%20Access.md)
    - [内部可访问](./compilers/CSharp/Internal%20Access.md)
    - [多继承查找时的歧义](./compilers/CSharp/Multiple%20Inheritance%20Lookup%20Ambuguity.md)
    - [解析重载](./compilers/CSharp/Overload%20Resolution.md)
    - [源代码与元数据的冲突](./compilers/CSharp/Source-Metadata%20Conflict.md)
    - [`static`类型约束](./compilers/CSharp/Static%20Type%20Constraints.md)
    - [System.TypedReference](./compilers/CSharp/System.TypedReference.md)
    - [Unicode 版本](./compilers/CSharp/Unicode%20Version.md)
    - [Win32 资源](./compilers/CSharp/Win32%20Resources.md)
    - [参数列表](./compilers/CSharp/__arglist.md)

  - [设计](./compilers/Design/README.md)
    - [关闭转换](./compilers/Design/ClosureConversion.md)
    - [转换器](./compilers/Design/Parser.md)
    - [意外情况](./compilers/Design/UnexpectedConditiond.md)

  - 动态分析
    - [元数据格式](./compilers/Dynamic%20Analysis/MetadataFormat.md)

  - Visual Basic
    - [与Enum成员冲突](./compilers/Visual%20Basic/Clashing%20Enum%20Members.md)
    - [命令行](./compilers/Visual%20Basic/CommandLine.md)
    - [编译器变化-VS2015](./compilers/Visual%20Basic/Compiler%20Breaking%20Changes%20-%20VS2015.md)
    - [编译器变化- POST VS2017](./compilers/Visual%20Basic/Compiler%20Breaking%20Changes%20-%20post%20VS2017.md)

  - [生成 Core CLR 编译器](./compilers/BuildingforCoreCLR.md)
  - [共置核心类型](./compilers/Co-locatedCoreTypes.md)
  - [确定性输入](./compilers/DeterministicInputs.md)
  - [错误日志格式](./compilers/ErrorLogFormat.md)
  - [在别名组件中的扩展方法](./compilers/ExtensionMethodsinAliasedAssemblies.md)
  - [交互环境中导入](./compilers/InteractiveImports.md)
  - [规则集格式](./compilers/RuleSetFormat.md)

- 贡献
  - [在 Unix 上生成、编译和调试](./contributing/Building%2C%20Debugging%2C%20and%20Testing%20on%20Unix.md)
  - [在 Windows 上生成、编译和调试](./contributing/Building%2C%20Debugging%2C%20and%20Testing%20on%20Windows.md)
  - [编译器测试计划](./contributing/Compiler%20Test%20Plan.md)
  - [开发一个语言特性](./contributing/Developing%20a%20Language%20Feature.md)
  - [IDE 测试计划](./contributing/IDE%20Test%20Plan.md)
  - [Powershell 原则](./contributing/Powershell%20Guidelines.md)

- 设计笔记
    - [2015-01-14 VB Design Meeting](./designNotes/2015-01-14%20VB%20Design%20Meeting.md)
    - [2015-01-21 C# Design Meeting](./designNotes/2015-01-21%20C%23%20Design%20Meeting.md)
    - [2015-01-28 C# Design Meeting](./designNotes/2015-01-28%20C%23%20Design%20Meeting.md)
    - [2015-02-04 C# Design Meeting](./designNotes/2015-02-04%20C%23%20Design%20Meeting.md)
    - [2015-02-11 C# Design Meeting](./designNotes/2015-02-11%20C%23%20Design%20Meeting.md)
    - [2015-03-04 C# Design Meeting](./designNotes/2015-03-04%20C%23%20Design%20Meeting.md)
    - [2015-03-10,17 C# Design Meeting](./designNotes/2015-03-10%2C17%20C%23%20Design%20Meeting.md)
    - [2015-03-18 C# Design Meeting](./designNotes/2015-03-18%20C%23%20Design%20Meeting.md)
    - [2015-03-24 C# Design Meeting](./designNotes/2015-03-24%20C%23%20Design%20Meeting.md)
    - [2015-03-25 C# Design Meeting](./designNotes/2015-03-25%20C%23%20Design%20Meeting.md)
    - [2015-03-25 C# Design Review](./designNotes/2015-03-25%20C%23%20Design%20Review.md)
    - [2015-04-01,8 C# Design Meeting](./designNotes/2015-04-01%2C8%20C%23%20Design%20Meeting.md)
    - [2015-04-14 C# Design Meeting](./designNotes/2015-04-14%20C%23%20Design%20Meeting.md)
    - [2015-04-15 C# Design Meeting](./designNotes/2015-04-15%20C%23%20Design%20Meeting.md)
    - [2015-04-22 C# Design Review](./designNotes/2015-04-22%20C%23%20Design%20Review.md)
    - [2015-05-20 C# Design Meeting](./designNotes/2015-05-20%20C%23%20Design%20Meeting.md)
    - [2015-05-25 C# Design Meeting](./designNotes/2015-05-25%20C%23%20Design%20Meeting.md)
    - [2015-07-01 C# Design Meeting](./designNotes/2015-07-01%20C%23%20Design%20Meeting.md)
    - [2015-07-07 C# Design Meeting](./designNotes/2015-07-07%20C%23%20Design%20Meeting.md)
    - [2015-08-18 C# Design Meeting](./designNotes/2015-08-18%20C%23%20Design%20Meeting.md)
    - [2015-09-01 C# Design Meeting](./designNotes/2015-09-01%20C%23%20Design%20Meeting.md)
    - [2015-09-02 C# Design Meeting](./designNotes/2015-09-02%20C%23%20Design%20Meeting.md)
    - [2015-09-08 C# Design Meeting](./designNotes/2015-09-08%20C%23%20Design%20Meeting.md)
    - [2015-10-07 C# Design Review Notes](./designNotes/2015-10-07%20C%23%20Design%20Review%20Notes.md)
    - [2015-11-02 C# Design Demo](./designNotes/2015-11-02%20C%23%20Design%20Demo.md)
    - [2016-02-29 C# Design Meeting](./designNotes/2016-02-29%20C%23%20Design%20Meeting.md)
    - [2016-04-06 C# Design Meeting](./designNotes/2016-04-06%20C%23%20Design%20Meeting.md)

- 功能特性
  - [ExperimentalAttribute](./features/ExperimentalAttribute.md)
  - [async-main](./features/async-main.md)
  - [async-main.test](./features/async-main.test.md)
  - [解构](./features/deconstruction.md)
  - [废弃变量](./features/discards.md)
  - [动态分析](./features/dynamic-analysis.work.md)
  - [源代码生成器](./features/generators.md)
  - [generators.work](./features/generators.work.md)
  - [localfuntions](./features/local-functions.md)
  - [localfuntions.test](./features/local-functions.test.md)
  - [localfuntions.work](./features/local-functions.work.md)
  - [records](./features/records.md)
  - [records.work](./features/records.work.md)
  - [refout](./features/refout.md)
  - [sdk](./features/sdk.md)
  - [task 类型](./features/task-types.md)
  - [抛出异常](./features/throwexpr.md)
  - [元组](./features/tuples.md)
  - [wildcards.work](./features/wildcards.work.md)

- ide
  - [词汇表](./ide/glossary.md)

- 基础结构
  - [跨平台](./infrastructure/cross-platform.md)
  - [特性分支 Jenkins](./infrastructure/feature%20branches%20jenkins.md)
  - [msbuild](./infrastructure/msbuild.md)

- 规范
  - CSharp 6
    - [更好的更好处](./specs/CSharp%206/Better%20Betterness.md)
  - [PortablePdb元数据](./specs/PortablePdb-Metadata.md)

- [Public Api 增加可选参数](./Adding%20Optional%20Parameters%20in%20Public%20API.md)
- [API 大的变动](./Breaking%20API%20Changes.md)
- [语言特性状态](./Language%20Feature%20Status.md)