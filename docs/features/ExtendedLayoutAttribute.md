System.Runtime.InteropServices.ExtendedLayoutAttribute 编译器支持
=======================================================================

.NET 运行时团队正在引入一个新特性 `System.Runtime.InteropServices.ExtendedLayoutAttribute`，该特性将允许运行时团队为类型提供额外的布局选项，主要用于互操作场景。
为了提供运行时团队所要求的用户体验，C# 和 VB 编译器将对该特性提供以下支持：

- 如果某个类型应用了 `ExtendedLayoutAttribute`，编译器将在该类型的 `TypeAttributes` 标志中生成 `TypeAttributes.ExtendedLayout` 值。
- 如果某个类型应用了 `ExtendedLayoutAttribute`，编译器将不允许对该类型应用 `StructLayoutAttribute`。
- （仅限 C# 编译器）如果某个类型应用了 `ExtendedLayoutAttribute`，编译器将不允许对该类型应用 `InlineArrayAttribute`。
- 在 Roslyn 编译器中，应用了 `ExtendedLayoutAttribute` 的类型的 `ITypeSymbol` 将具有以下行为：
  - `Layout` 属性将返回一个 `TypeLayout` 实例，其中 `LayoutKind` 设置为 `Extended`（`1`），`Size` 设置为 `0`，`Pack` 设置为 `0`。
- 嵌入到程序集中的类型（使用 `NoPia` 技术）将在嵌入类型上保留 `ExtendedLayoutAttribute`。

Roslyn 编译器不会了解 `ExtendedLayoutAttribute` 上可用的具体选项，因为这些选项将由运行时团队定义，并可能随时间扩展。编译器不会尝试检测特定布局选项的无效字段类型。运行时团队未来可能会添加分析器来处理这些场景。

编译器仅识别该特性的存在及其对类型符号的 `LayoutKind` 和生成特性的影响。
