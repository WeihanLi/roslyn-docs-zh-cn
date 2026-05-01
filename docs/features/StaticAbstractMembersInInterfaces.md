接口中的静态抽象成员
=====================================

接口可以指定抽象静态成员，实现该接口的类和结构体则需要提供显式或隐式实现。这些成员可以通过受该接口约束的类型参数来访问。

提案：
- https://github.com/dotnet/csharplang/issues/4436
- https://github.com/dotnet/csharplang/blob/main/proposals/static-abstracts-in-interfaces.md

功能分支：https://github.com/dotnet/roslyn/tree/features/StaticAbstractMembersInInterfaces

测试计划：https://github.com/dotnet/roslyn/issues/52221
