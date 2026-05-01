已检查的用户定义运算符
=====================================

C# 应支持为以下用户定义运算符定义 `checked` 变体，以便用户可以根据需要选择启用或禁用溢出行为：
*  `++` 和 `--` 一元运算符（https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#postfix-increment-and-decrement-operators 和 https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#prefix-increment-and-decrement-operators）。
*  `-` 一元运算符（https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#unary-minus-operator）。
*  `+`、`-`、`*` 和 `/` 二元运算符（https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#arithmetic-operators）。
*  显式转换运算符。

提案：
- https://github.com/dotnet/csharplang/issues/4665
- https://github.com/dotnet/csharplang/blob/main/proposals/checked-user-defined-operators.md

功能分支：https://github.com/dotnet/roslyn/tree/features/CheckedUserDefinedOperators

测试计划：https://github.com/dotnet/roslyn/issues/59458
