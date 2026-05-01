弃元
--------

弃元是可以赋值但无法读取的变量。它们没有名称，而是用 `_`（下划线）表示。
在 C#7.0 中，它们可以出现在以下上下文中：

- out 变量声明，如 `bool found = TryGetValue(out var _)` 或 `bool found = TryGetValue(out _)`
- 解构赋值，如 `(x, _) = deconstructable;`
- 解构声明，如 `(var x, var _) = deconstructable;`
- is 模式，如 `x is int _`
- switch/case 模式，如 `case int _:`

弃元的主要表示形式是声明表达式中的 `_`（下划线）指定符。例如，out 变量声明中的 `int _` 或解构声明中的 `var (_, _, x)`。

弃元的第二种表示形式是将表达式 `_` 作为 `var _` 的简写，当作用域中没有名为 `_` 的变量时使用。它允许在 out var、解构赋值和声明，以及普通赋值（`_ = IgnoredReturn();`）中使用。但在 C#7.0 的模式中不允许使用。
当作用域中确实存在名为 `_` 的变量时，表达式 `_` 就是对该变量的普通引用，与旧版 C# 中相同。

### 语法变更

```ANTLR
declaration_expression
: type variable_designation
;

variable_designation
: single_variable_designation
| parenthesized_variable_designation
| discard_designation // new
;

discard_designation // new
: '_'
;
```

**参考资料**
[C# 语言设计说明 2016 年 10 月 25 日和 26 日](https://github.com/dotnet/roslyn/issues/16640)
[C# 设计说明 2016 年 10 月 18 日](https://github.com/dotnet/roslyn/issues/16482)
[弃元设计提案](https://github.com/dotnet/roslyn/issues/14862)
注意：这些说明中曾将弃元称为"通配符"。
