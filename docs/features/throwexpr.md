## throw 表达式

我们扩展了表达式形式集，添加了以下内容：

```antlr
throw_expression
    : 'throw' null_coalescing_expression
    ;

null_coalescing_expression
    : throw_expression
    ;
```

类型规则如下：

- *throw_expression* 没有类型。
- *throw_expression* 可通过隐式转换转换为任意类型。

流分析规则如下：

- 对于每个变量 *v*，*v* 在 *throw_expression* 的 *null_coalescing_expression* 之前确定赋值，当且仅当它在 *throw_expression* 之前确定赋值。
- 对于每个变量 *v*，*v* 在 *throw_expression* 之后确定赋值。

*throw 表达式*仅在以下语法上下文中允许：
- 作为三元条件运算符 `?:` 的第二或第三操作数
- 作为空合并运算符 `??` 的第二操作数
- 作为表达式体 lambda 或方法的主体

> 注意：throw 表达式的其余语义与当前语言规范中 *throw_statement* 的语义相同。
