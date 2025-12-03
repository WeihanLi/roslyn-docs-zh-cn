> 本功能设计和实现的开放问题可以在 [patterns.work.md](patterns.work.md) 中找到。

C# 模式匹配
=======================

C# 的模式匹配扩展能够实现代数数据类型和函数式语言模式匹配的许多优点，但以一种与底层语言感觉平滑集成的方式。基本功能是：[记录类型](records.md)，其语义意义由数据的形状描述（作为单独的功能处理）；以及模式匹配，这是一种能够对这些数据类型进行极其简洁的多级分解的新形式。这种方法的元素受到编程语言 [F#](http://www.msr-waypoint.net/pubs/79947/p29-syme.pdf "可扩展模式匹配通过轻量级语言") 和 [Scala](http://lampwww.epfl.ch/~emir/written/MatchingObjectsWithPatterns-TR.pdf "用模式匹配对象") 中相关功能的启发。

## Is 表达式

`is` 运算符被扩展为针对*模式*测试表达式。

```antlr
is_pattern_expression
    : relational_expression 'is' pattern
    ;

relational_expression
    : is_pattern_expression
    ;
```

这种形式的 *relational_expression* 是 C# 规范中现有形式的补充。如果 `is` 标记左侧的 *relational_expression* 不指定值或没有类型，则是编译时错误。

作为 *is_pattern_expression* 右侧出现的 *constant_pattern* 在语法上被限制为 *shift_expression*，即使其他地方出现的 *constant_pattern* 在语法上可以是任何 *expression*。为简单起见，该限制未在语法中显示。

模式的每个*标识符*引入一个新的局部变量，该变量在 `is` 运算符为 `true` 后被*明确赋值*（即*为真时明确赋值*）。

> 注意：`is-expression` 中的 *type* 和 *is_pattern_expression* 中的 *constant_pattern* 之间在技术上存在歧义，两者都可能是限定标识符的有效解析。我们尝试将其绑定为类型以与语言的早期版本兼容；只有在失败时才将其解析为常量模式。

## 模式

模式用于 `is` 运算符和 *switch_statement* 中，以表达要与传入数据进行比较的数据形状。模式可以是递归的，以便数据的部分可以与子模式匹配。

```antlr
pattern
    : declaration_pattern
    | constant_pattern
    | deconstruction_pattern
    | property_pattern
    | discard_pattern
    | var_pattern
    ;

declaration_pattern
    : type simple_designation
    ;

constant_pattern
    : constant_expression
    ;

property_pattern
    : type? property_subpattern simple_designation?
    ;

property_subpattern
    : '{' subpatterns? '}'
    ;

subpatterns
    : subpattern
    | subpattern ',' subpatterns
    ;

subpattern
    : pattern
    | identifier ':' pattern
    ;

deconstruction_pattern
    : type? '(' subpatterns? ')' property_subpattern? simple_designation?
    ;

simple_designation
    : single_variable_designation
    | discard_designation
    ;

discard_pattern
    : discard_designation
    ;

var_pattern
    : 'var' designation
    ;
```

如果 _property_pattern_ 的任何 _subpattern_ 不包含 _identifier_（它必须是第二种形式，具有 _identifier_），则是语义错误。

没有特殊的"空检查模式"，因为检查空值的能力作为简单属性模式的特殊情况而存在。要检查字符串 `s` 是否非空，可以编写以下任何形式

``` c#
if (s is object o) ... // o 是 object 类型
if (s is string x) ... // x 是 string 类型
if (s is {} x) ... // x 是 string 类型
if (s is {}) ...
```

每个模式都针对假设的*输入操作数*进行绑定。在 *is_pattern_expression* 的情况下，这是左侧的值。在 `switch` 语句分支的情况下，它是 switch 表达式。在嵌套模式的情况下，它是从封闭对象中提取相关值的结果。

### 类型模式

*type_pattern* 既测试输入操作数是否持有给定类型的值，又在测试成功时将其转换为该类型。这引入了一个由 *simple_designation* 命名的给定类型的局部变量。当模式匹配操作的结果为真时，该局部变量被*明确赋值*。

```antlr
type_pattern
    : type simple_designation
    ;
```

此表达式的运行时语义是它针对模式中的 *type* 测试输入操作数的运行时类型。如果它是该运行时类型（或某个子类型），则 `is 运算符` 的结果为 `true`。如果 *simple_designation* 是 *single_variable_designation*，则它声明一个由 *identifier* 命名的新局部变量，当结果为 `true` 时被赋予输入操作数的值。

左侧的静态类型和给定类型的某些组合被认为是不兼容的，会导致编译时错误。如果存在从 `E` 到 `T` 的标识转换、隐式引用转换、装箱转换、显式引用转换或拆箱转换，则静态类型为 `E` 的值被称为与类型 `T` *模式兼容*。如果类型为 `E` 的表达式与其匹配的类型模式中的类型不兼容，则是编译时错误。

> 注意：这对于类型参数不太正确，它们应该始终被认为是模式兼容的。

类型模式对于执行引用类型的运行时类型测试很有用，并替换了以下习惯用法

```cs
Type v;
if (value is Type)
{
    v = (Type)value;
}
```

或

``` c#
var v = expr as Type;
if (v != null) { // 使用 v 的代码 }
```

改为更简洁的

```cs
if (expr is Type v) { // 使用 v 的代码 }
```

如果 *type* 是可空值类型，则是错误。

类型模式可用于测试可空类型的值：类型为 `Nullable<T>`（或装箱的 `T`）的值与类型模式 `T2 id` 匹配，如果该值非空且 `T2` 的类型是 `T`，或 `T` 的某个基类型或接口。例如，在代码片段中

```cs
int? x = 3;
if (x is int v) { // 使用 v 的代码 }
```

`if` 语句的条件在运行时为 `true`，变量 `v` 持有类型为 `int` 的值 `3`。

### 常量模式

常量模式针对常量值测试表达式的值。常量可以是任何常量表达式，如文字、声明的 `const` 变量的名称或枚举常量。

如果 *e* 和 *c* 都是整型，则如果表达式 `e == c` 的结果为 `true`，则认为模式匹配。

否则，如果 `object.Equals(c, e)` 返回 `true`，则认为模式匹配。在这种情况下，如果 *e* 的静态类型与常量的类型不*模式兼容*，则是编译时错误。

```antlr
constant_pattern
    : constant_expression
    ;
```

### Var 模式

``` antlr
var_pattern
    : 'var' designation
    ;
```

输入操作数总是与模式 `var identifier` 匹配。换句话说，与带有 *simple_designator* 的 *var pattern* 的匹配总是成功。在运行时，输入操作数的值绑定到一个新引入的局部变量。局部变量的类型是输入操作数的静态类型。

如果在使用 *var_pattern* 的地方名称 `var` 绑定到类型，则是错误。

> 注意：我们需要描述当 *designation* 是 *tuple_designation* 时的语义。

### 丢弃模式

输入操作数总是与丢弃 `_` 匹配。换句话说，每个表达式都与丢弃模式匹配。

### 属性模式

``` antlr
property_pattern
	: type? property_subpattern simple_designation?
	;

property_subpattern
	: '{' subpatterns? '}'
	;

subpatterns
	: subpattern
	| subpattern ',' subpatterns
	;

subpattern
	: pattern
	| identifier ':' pattern
	;
```

属性模式测试输入操作数是否是给定类型的实例，还测试其某些可访问属性或字段是否与给定的子模式匹配。

要测试的类型选择如下：
- 如果 *property_pattern* 有 *type* 部分，则那是要测试的类型。该类型不能是可空值类型。
- 否则，如果输入操作数不是可空值类型，则使用其静态类型。
- 否则使用输入操作数类型的基础类型。

*simple_designation*，如果它是 *single_variable_designation*，则命名此类型的新引入变量。

*property_subpattern* 中出现的每个 *subpattern* 指定要检查的属性或字段。子模式必须是第二种形式，带有 *identifier*。*identifier* 必须命名要测试的类型的可访问且可读的实例属性或字段。

在运行时，当该属性或字段的值作为 *subpattern* 模式的输入操作数处理时匹配，则 *subpattern* 被*满足*。

如果类型测试成功且所有子模式都满足，则属性模式匹配。子模式匹配的顺序未指定，失败的匹配可能不会在运行时测试所有子模式。

### 解构模式

解构模式类似于 *property_pattern*，但涉及 *tuple* 的解构或用户定义的 *Deconstruct* 方法的调用。

``` antlr
deconstruction_pattern
	: type? '(' subpatterns? ')' property_subpattern? simple_designation?
	;
```

如果解构模式有单个子模式但省略了类型，则是错误。

要测试的类型按照 *property_pattern* 中确定。该类型必须是 *cardinality* 与括号之间的 *subpatterns* 数量相同的元组类型，或者它必须是包含唯一 `Deconstruct` 方法的类型，该方法具有那么多 `out` 参数，如 *deconstruction assignment* 功能所定义。

如果从输入操作数的解构中为该位置检索的值作为相应模式的输入操作数处理时匹配，则括号内的 *deconstruction_pattern* 中的子模式被满足。如果这样的子模式有 *identifier*，如果它不是相应元组元素或 `Deconstruct` `out` 参数的名称，则是编译时错误。

如果类型测试成功且所有子模式都满足，则解构模式匹配。子模式匹配的顺序未指定，失败的匹配可能不会在运行时测试所有子模式。

> 注意：此规范尚未处理用于位置匹配的 `ITuple`。
> 

## Switch 表达式

添加了 *switch_expression* 以支持表达式上下文的 `switch` 类语义。

C# 语言语法通过以下语法产生式进行增强：


``` antlr
switch_expression
    : null_coalescing_expression switch '{' switch_expression_case_list '}'
    ;
switch_expression_case_list
    : switch_expression_case
    | switch_expression_case_list ',' switch_expression_case
    ;
switch_expression_case
    : pattern where_clause? '=>' expression
    ;
```

*switch_expression* 不允许作为 *expression_statement*。

*switch_expression* 的类型是 *switch_expression_case* 的 `=>` 标记右侧出现的表达式的*最佳公共类型*。

如果编译器可以证明（使用一组尚未指定的技术）某些 *switch_section* 的模式不会影响结果，因为某些先前的模式将始终匹配，则是错误。

在运行时，*switch_expression* 的结果是第一个 *switch_expression_case* 的 *expression* 的值，对于该 *switch_expression_case*，*switch_expression* 左侧的表达式与 *switch_expression_case* 的模式匹配，并且 *switch_expression_case* 的 *where_clause* 的表达式（如果存在）求值为 `true`。

> 注意：我们需要指定当情况集不完整时，编译时和运行时会发生什么。

## 模式匹配的一些示例

### 算术简化

假设我们定义了一组递归类型来表示表达式（根据单独的提案）：

```cs
abstract class Expr;
class X() : Expr;
class Const(double Value) : Expr;
class Add(Expr Left, Expr Right) : Expr;
class Mult(Expr Left, Expr Right) : Expr;
class Neg(Expr Value) : Expr;
```

现在我们可以定义一个函数来计算表达式的（未简化的）导数：

```cs
Expr Deriv(Expr e)
{
  switch (e) {
    case X(): return Const(1);
    case Const(_): return Const(0);
    case Add(var Left, var Right):
      return Add(Deriv(Left), Deriv(Right));
    case Mult(var Left, var Right):
      return Add(Mult(Deriv(Left), Right), Mult(Left, Deriv(Right)));
    case Neg(var Value):
      return Neg(Deriv(Value));
  }
}
```

表达式简化器演示位置模式：

```cs
Expr Simplify(Expr e)
{
  switch (e) {
    case Mult(Const(0), _): return Const(0);
    case Mult(_, Const(0)): return Const(0);
    case Mult(Const(1), var x): return Simplify(x);
    case Mult(var x, Const(1)): return Simplify(x);
    case Mult(Const(var l), Const(var r)): return Const(l*r);
    case Add(Const(0), var x): return Simplify(x);
    case Add(var x, Const(0)): return Simplify(x);
    case Add(Const(var l), Const(var r)): return Const(l+r);
    case Neg(Const(var k)): return Const(-k);
    default: return e;
  }
}
```
