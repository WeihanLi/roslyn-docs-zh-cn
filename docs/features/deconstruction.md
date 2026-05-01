解构快速入门指南（C# 7.0）
----------------------------------------------
1. 安装 Visual Studio 2017
2. 启动一个 C# 项目
3. 从 NuGet 添加对 `System.ValueTuple` 包的引用
![安装 ValueTuple 包](img/install-valuetuple.png)
4. 使用解构：

```C#
public class C
{
        public static void Main()
        {
                
              int code;
              string message;

              var pair = (42, "hello");
              (code, message) = pair; // deconstruct a tuple into existing variables
              Console.Write(message); // hello

              (code, message) = new Deconstructable(); // deconstruct any object with a proper Deconstruct method into existing variables
              Console.Write(message); // world
              
              (int code2, string message2) = pair; // deconstruct into new variables
              var (code3, message3) = new Deconstructable(); // deconstruct into new 'var' variables
        }
}

public class Deconstructable
{
        public void Deconstruct(out int x, out string y)
        {
                x = 43;
                y = "world";
        }
}
```

设计
------
本设计文档将涵盖两种解构：解构到已有变量（*解构赋值*）和解构到新变量（*解构声明*）。

以下是解构赋值的示例：
```C#
class C
{
    static void Main()
    {
        long x;
        string y;

        (x, y) = new C();
        System.Console.WriteLine(x + "" "" + y);
    }

    public void Deconstruct(out int a, out string b)
    {
        a = 1;
        b = ""hello"";
    }
}
```

### 解构赋值（解构到已有变量）：

这不引入任何语言语法变更。我们有一个*赋值表达式*（在 C# 语法中也简称为*赋值*），其中*一元表达式*（左侧）是一个包含可赋值值的*元组表达式*。
简而言之，在一般情况下，这会在赋值右侧的表达式上查找 `Deconstruct` 方法，用适当数量的 `out var` 参数调用它，将这些输出值（如需要）转换后赋值给左侧的变量。但在右侧表达式是元组（元组表达式或元组类型）的特殊情况下，元组的元素直接赋值给左侧变量，无需调用 Deconstruct。

如果左侧是嵌套的，该过程将重复进行。例如，在 `(x, (y, z)) = deconstructable;` 中，`deconstructable` 将被解构为两部分，第二部分将进一步解构。

在右侧表达式是元组表达式的情况下，它首先被赋予类型。因此在 `long x; string y; (x, y) = (1, null);` 中，右侧的字面量在解构开始之前就被类型化为 `long` 和 `string`，这意味着在解构步骤期间不需要转换。

我们已经注意到，元组（作为 `System.ValueTuple` 底层类型的语法糖）不需要调用 `Deconstruct`。
.NET 框架还包含一组 `System.Tuple` 类型。这些类型不被识别为 C# 元组，因此将依赖 *Deconstruct* 模式。这些 `Deconstruct` 方法将作为 `System.Tuple` 的扩展方法提供，最多支持 3 层嵌套（即 21 个元素）。

*解构赋值*返回一个元组值（元素使用默认名称），其形状和类型与左侧相同，并保存解构所得的（已转换）部分。

#### 求值顺序

求值顺序可以总结为：(1) 左侧的所有副作用，(2) 所有 `Deconstruct` 调用（如果不是元组），(3) 转换（如需要），以及 (4) 赋值。

在一般情况下，解构赋值的降级会将 `(expressionX, expressionY, expressionZ) = expressionRight` 转换为：

```
// do LHS side-effects
tempX = &evaluate expressionX
tempY = &evaluate expressionY
tempZ = &evaluate expressionZ

// do Deconstruct
evaluate right and evaluate Deconstruct in three parts (tempA, tempB and tempC)

// do conversions
tempConvA = convert tempA
tempConvB = convert tempB
tempConvC = convert tempC

// do assignments
tempX = tempConvA
tempY = tempConvB
tempZ = tempConvC
```

嵌套 `(x, (y, z))` 的求值顺序为：
```
// do LHS side-effects
tempX = &evaluate expressionX
tempY = &evaluate expressionY
tempZ = &evaluate expressionZ

// do Deconstruct
evaluate right and evaluate Deconstruct into two parts (tempA and tempNested)
evaluate Deconstruct on tempNested intwo two parts (tempB and tempC)

// do conversions
tempConvA = convert tempA
tempConvB = convert tempB
tempConvC = convert tempC

// do assignments
tempX = tempConvA
tempY = tempConvB
tempZ = tempConvC
```

最简单情况（局部变量、字段、数组索引器或任何返回 ref 的情况）且无需转换时的求值顺序：
```
evaluate side-effect on the left-hand-side variables
evaluate Deconstruct passing the references directly in
```

#### Deconstruct 方法的解析

解析等价于以适当数量的参数类型化 `rhs.Deconstruct(out var x1, out var x2, ...);`。
它基于正常的重载决策。
这意味着 `rhs` 不能是 dynamic，且 `Deconstruct` 方法的参数不能是类型参数。不会找到 `Deconstruct<T>(out T x1, out T x2)` 方法。
此外，`Deconstruct` 方法必须是实例方法或扩展方法（但不能是静态方法）。它还必须返回 `void`。


### 解构声明（解构到新变量）：

*解构声明*也用*赋值*表示，但左侧是*声明表达式*或包含*声明表达式*的元组表达式。
*解构声明*可以看作两个步骤：(1) 声明新的局部变量，(2) 将*解构赋值*应用到这些局部变量。
赋值左侧声明新局部变量可以有多种形式。最简单的情况是 `(int x, string y)`，即包含声明表达式的元组表达式。变体包括嵌套声明如 `(int x, (string y, long z))`（声明 3 个局部变量）和隐式类型声明如 `(var x, var y)`。后者也可以使用简写 `var (x, y)` 来写，这是一个带括号指定的声明表达式。
`var` 是唯一允许使用此简写的情况（因此 `int (x, y)` 是不合法的）。

与*解构赋值*的情况一样，右侧的元组表达式从左侧推断类型。对于*解构声明*，情况也相同，但如果左侧的任何类型是 `var`，则使用右侧对应元素的自然类型。
例如，在 `(string x, byte y, var z) = (null, 1, 2);` 中，`null` 的类型是 `string`，字面量 `1` 的类型是 `byte`（从 `y` 推断），字面量 `2` 的类型是 `int`（其自然类型）。

在 C#7.0 中，*解构声明*只允许作为顶级语句。它们不允许混合声明表达式和可赋值表达式（如 `(existing, var declared) = (1, 2)`）。

### 语法变更

```ANTLR
expression
: ... // existing
| declaration_expression // new (only allowed in C#7.0 in certain contexts, such as out var, deconstruction and pattern declarations)
;

declaration_expression // new
: type variable_designation
;

variable_designation // new
: single_variable_designation
| parenthesized_variable_designation
| discard_designation
;

single_variable_designation // new
: identifier
;

parenthesized_variable_designation // new
: '(' variable_designation (',' variable_designation)+ ')'
;

discard_designation // new
: '_'
;

foreach_variable_statement // new
    : 'foreach' '(' declaration_expression 'in' expression ')' embedded_statement
    ;
```

**参考资料**

[C# 2016 年 10 月 25-26 日设计说明](https://github.com/dotnet/csharplang/blob/main/meetings/2016/LDM-2016-10-25-26.md)

[C# 2016 年 9 月 6 日设计说明](https://github.com/dotnet/csharplang/blob/main/meetings/2016/LDM-2016-09-06.md)

[C# 2016 年 7 月 13 日设计说明](https://github.com/dotnet/csharplang/blob/main/meetings/2016/LDM-2016-07-13.md)

[C# 2016 年 5 月 3-4 日设计说明](https://github.com/dotnet/csharplang/blob/main/meetings/2016/LDM-2016-05-03-04.md)

[C# 2016 年 4 月 12-22 日设计说明](https://github.com/dotnet/csharplang/blob/main/meetings/2016/LDM-2016-04-12-22.md)

[声明表达式设计](https://github.com/dotnet/csharplang/issues/365)

[C# 7.0 新功能](https://blogs.msdn.microsoft.com/dotnet/2016/08/24/whats-new-in-csharp-7-0)文章中有关于解构的章节。

**可能的未来扩展**
- 在 [let 和 from 子句](https://github.com/dotnet/csharplang/issues/189)中使用解构
- 在 [lambda 参数列表](https://github.com/dotnet/csharplang/issues/258)中使用解构
- [解构模式](https://github.com/dotnet/csharplang/issues/277)
- 允许[混合赋值和声明](https://github.com/dotnet/csharplang/issues/125)的解构，从而也允许在表达式上下文中使用解构声明

更新的提案和讨论请参见 [C# Lang](https://github.com/dotnet/csharplang) 仓库。
