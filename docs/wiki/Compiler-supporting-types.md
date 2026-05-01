本页面记录了向编译器中添加新 BCL 类型的一些最佳实践。

## 特性（Attributes）

如果层级允许，新的**特性**应使用 `Obsolete` 特性标记（将 `error` 设置为 `true`，使其不允许出现在源代码中），并标记 `EditorBrowsable(EditorBrowsableState.Never)`。

例如，尽管我们当时没有这样做，但 `TupleElementNames` 特性最好以这种方式标记。这样，当使用 Visual Studio 2015 实现带有元组名称的 C# 7.0 接口时，IDE 就不会在源代码中引入 `TupleElementNames`（一旦升级到 Visual Studio 2017，这会造成问题）。

## 类型

新**类型**最好先添加到 corlib（CoreCLR、CoreRT、Mono、Desktop）。然后可以提供一个独立包，包含针对旧目标的下层实现以及针对新目标的类型转发。

先提供包，然后再将类型迁移到 corlib 作为第二步，这会带来麻烦、产生歧义窗口和出错机会。参见 ValueTuple 库工作的[日志](https://github.com/dotnet/roslyn/issues/13177)。如果我们选择这样做，应尽快更新该包并将其标记为*正式版*（非预发布版）。

## 与旧版编译器的往返兼容

使用旧版客户端项目（使用旧版编译器）来使用新库（由新编译器生成，含有新元数据）时，存在一个普遍问题。每个引入此类元数据的特性都需要在以下几种方式之间做出选择：

1. 污染元数据（使旧版编译器拒绝它）
2. 让客户端面临破坏（当将项目升级到新版编译器时）
3. 确保新编译器在遇到它不理解的元数据时，维持旧版编译器的行为

`ref readonly` 特性选择了方式（1）。

目前有关于调整编译器对缺失元组名称的容忍度的讨论（采用第三种方式，而非第二种）：https://github.com/dotnet/roslyn/issues/20528。

## 历史记录：
- [.NET Framework 4.7.1](https://blogs.msdn.microsoft.com/dotnet/2017/09/28/net-framework-4-7-1-runtime-and-compiler-features/) 中的变更（添加了 ref readonly 和 ref struct 的特性，使 `ValueTuple` 类型可序列化，添加了 ITuple，添加了 `RuntimeFeature` API）
- [.NET Framework 4.7](https://blogs.msdn.microsoft.com/dotnet/2017/04/05/announcing-the-net-framework-4-7/) 中的变更（添加了 `ValueTuple` 类型）

## 待解决问题：
- 验证当特性被标记为过时且 IDE 尝试从你正在重写的方法中复制它时，IDE 当前的行为。
