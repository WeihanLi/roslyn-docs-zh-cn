# 编译器测试计划

本文档提供了关于语言交互和测试编译器更改的思考指导。

## 一般关注点

- 规范作为测试指南的完整性（规范是否足够完整，可以建议编译器在每种情况下应该做什么？）
- *Ping* 新的破坏性更改和对合作伙伴团队的一般 ping（Bill、Kathleen、Mads、IDE、Razor）
- 帮助审查外部文档
- 向后和向前兼容性（与以前和未来编译器的互操作，双向）
- 错误处理/恢复（缺少库，包括 mscorlib 中缺少的类型；解析错误、模糊查找、不可访问查找、找到错误类型的东西、找到实例与静态的东西、上下文的错误类型、值与变量）
- BCL（包括 mono）和其他客户影响
- 确定性
- 从元数据加载（源代码与从元数据加载）
- 公共编译器 API（包括语义模型和下面列出的其他 API）：
    - GetDeclaredSymbol
    - GetEnclosingSymbol
    - GetSymbolInfo
    - GetSpeculativeSymbolInfo
    - GetTypeInfo
    - GetSpeculativeTypeInfo
    - GetMethodGroup
    - GetConstantValue
    - GetAliasInfo
    - GetSpeculativeAliasInfo
    - LookupSymbols
    - AnalyzeStatementsControlFlow
    - AnalyzeStatementControlFlow
    - AnalyzeExpressionDataFlow
    - AnalyzeStatementsDataFlow
    - AnalyzeStatementDataFlow
    - ClassifyConversion
    - GetOperation (`IOperation`)
    - GetCFG (`ControlFlowGraph`)，包括带有一些嵌套条件的场景
    - DocumentationCommentId API
- VB/F# 互操作
- C++/CLI 互操作（特别是元数据格式更改，例如 DIM、接口中的静态抽象成员或泛型属性）
- 性能和压力测试
- 可以构建 VS
- 检查 `Obsolete` 是否在绑定/降低中使用的成员上被遵守
- LangVersion
- IL 验证（根据需要在 `runtime` 仓库上提交问题并在[此处](https://github.com/dotnet/roslyn/issues/22872)跟踪）

- 该功能是否以任何方式使用加密哈希？（示例：文件本地类型的元数据名称、扩展类型、程序集强命名、PDB 文档表等）
    - 考虑改用非加密哈希，如 `XxHash128`。
    - 如果您必须在功能实现中使用加密哈希，请使用 `SourceHashAlgorithms.Default`，而不是任何特定的哈希。
    - 加密哈希绝不能包含在公共 API 名称中。更改默认加密算法将更改公共 API 表面，这将是极大的破坏性更改。
        - **不要**允许在字段、方法或类型名称中使用加密哈希的值
        - **允许**在属性或字段值中使用加密哈希的值
    - 任何时候编译器读取包含加密哈希的元数据时，即使是属性值，也要确保加密哈希算法名称包含在元数据中（例如，将其前缀到哈希值），以便随着时间的推移可以更改，编译器可以继续读取使用旧算法和新算法的元数据。

## 类型和成员

- 访问修饰符（public、protected、internal、protected internal、private protected、private）、static、ref
- 类型声明
  - 有或没有主构造函数的类
  - 有或没有位置成员的记录类/结构
  - 有或没有主构造函数的结构
  - 接口
  - 类型参数
- 文件本地类型
- 方法
  - 主构造函数
- 字段（必需和非必需）
- 属性（包括 get/set/init 访问器，必需和非必需）
- 事件（包括 add/remove 访问器）
- 参数修饰符：ref、out、in、ref readonly、params（用于数组，用于非数组）
- 属性（包括泛型属性和安全属性）
  - 编译器识别的属性不应在早期 LangVersion 中产生任何效果，
    除非在使用依赖于该属性的功能时应报告 LangVersion 错误
    （例如，InlineArray 到 Span 的转换）。
- 泛型（类型参数、协变性、约束包括 `class`、`struct`、`new()`、`unmanaged`、`notnull`、`allows ref struct`、具有可空性的类型和接口）
- 默认值和常量值
- 部分类
- 字面量
- 枚举（隐式与显式底层类型）
- 表达式树
- 迭代器
- 初始化器（对象、集合、字典）
- 数组（一维或多维、交错、初始化器、固定）
- 表达式体方法/属性/...
- 扩展方法
- 部分方法
- 命名参数和可选参数
- 字符串插值
- 原始字符串（包括插值）
- 属性（读写、只读、init-only、只写、自动属性、表达式体）
- 接口（隐式与显式接口成员实现）
- 委托
- 多声明
- NoPIA
- Dynamic
- ref 结构、readonly 结构
- 结构上的只读成员（方法、属性/索引器访问器、自定义事件访问器）
- SkipLocalsInit
- 带有 `where T : { class, struct, default }` 的方法重写或显式实现
- `extension` 块（使用基于内容的名称发出）

## 代码

- 运算符（参见下面 Eric 的列表）
- Lambda（捕获参数或局部变量、目标类型化）
- 执行顺序
- 目标类型化（var、lambda、整数）
- 转换（装箱/拆箱）
- 可空（包装、解包）
- 重载解析、override/hide/implement (OHI)
  - OverloadResolutionPriorityAttribute
- 继承（virtual、override、abstract、new）
- 匿名类型
- 元组类型和字面量（具有显式或推断名称的元素、长元组）、元组相等
- 范围字面量 (`1..2`) 和索引运算符 (`^1`)
- 解构
- 局部函数
- 不安全代码
- LINQ
- 构造函数、属性、索引器、事件、运算符和析构函数
- Async（类任务类型）和 async-iterator 方法
- 左值：合成字段是可变的
    - Ref / out 参数
    - 复合运算符（`+=`、`/=` 等）
    - 赋值表达式
- Ref 返回、ref readonly 返回、ref 三元、ref readonly 局部、ref 局部重新赋值、ref foreach
- Ref 字段
- `scoped` 参数和局部变量
- `struct` 构造函数中的 `this = e;`
- Stackalloc（包括初始化器）
- 模式（常量、声明、`var`、位置、属性和扩展属性、弃元、括号、类型、关系、`and`/`or`/`not`、列表、切片、常量 `string` 匹配 `Span<char>`）
- Switch 表达式
- With 表达式（在记录类和值类型上）
- 可空性注解（`?`、属性）和分析
- 确定赋值分析和自动默认结构字段
- 如果您添加了一个表达式可以出现在代码中的位置，请确保 `SpillSequenceSpiller` 处理它。使用 `switch` 表达式或 `stackalloc` 在该位置进行测试。
- 如果您添加了需要溢出的新表达式形式，请在 catch 过滤器中测试它。
- 如果您添加了可以嵌套在自身内部的新表达式，请测试深度嵌套它。可能会切断直接嵌套。
- 基于扩展的 Dispose、DisposeAsync、GetEnumerator、GetAsyncEnumerator、Deconstruct、GetAwaiter 等
- UTF8 字符串字面量（带有 'u8' 或 'U8' 类型后缀的字符串字面量）
- 内联数组元素访问和切片
- 集合表达式和展开元素

## 杂项

- 保留关键字（有时是上下文相关的）
- 预处理指令
- COM 互操作
- modopt 和 modreq
- CompilerFeatureRequiredAttribute
- CompilerLoweringPreserveAttribute
- ref 程序集
- extern 别名
- UnmanagedCallersOnly
- 遥测

## 与其他组件交互的测试

与 IDE、调试器和 EnC 的交互应与相关团队协调。一些要点：

- IDE
    - 着色和格式化
    - 输入体验和处理不完整代码
    - Intellisense（波浪线、点补全）
    - "转到"、查找所有引用和重命名
    - F1 帮助（用于新关键字）
    - cref 注释
    - UpgradeProject 代码修复程序
    - 更多：[IDE 测试计划](https://github.com/dotnet/roslyn/blob/main/docs/contributing/IDE%20Test%20Plan.md)

- 调试器 / EE
    - 单步执行、设置断点
    - 显示局部变量/自动窗口
        - 类型和值显示
        - 展开实例成员
        - 闭包类中的局部变量/参数/this
        - 局部变量和表达式上的属性（例如：[Dynamic]、[TupleElementNames]）
    - 在即时/监视窗口中编译表达式或悬停在表达式上
    - 在 [DebuggerDisplay("...")] 中编译表达式
    - 在局部变量/自动/监视窗口中赋值

- [编辑并继续](https://github.com/dotnet/roslyn/blob/main/docs/contributing/Testing%20for%20Interactive%20readiness.md)

- Live Unit Testing（检测）

- 与 VS 模板团队合作（如适用）
