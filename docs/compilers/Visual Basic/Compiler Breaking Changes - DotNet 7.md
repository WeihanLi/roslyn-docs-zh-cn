# 本文档列出了 .NET 6 到 .NET 7 之间 Roslyn Visual Basic 中已知的重大更改。

## `RequiredMembersAttribute` 不能手动应用

**在 VS 17.6 中引入**

作为实现 VB 对使用 `required` API 的支持的一部分，现在在源代码中手动应用 `RequiredMembersAttribute` 是一个错误。VB 现在将正确解释元数据中的这些属性，并允许实例化具有必要成员的类型。
