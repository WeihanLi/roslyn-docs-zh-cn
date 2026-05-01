# 重载解析

除了 C# 6 中 C# 语言规范的小量重载解析更改之外，Roslyn C# 编译器还实现了一些规范中没有的重载解析规则，以便与（C# 6 之前的）本机编译器兼容。这些更改由 `OverloadResolutionTests.cs` 和 `OperatorTests.cs` 中的单元测试支持。

以下是一些更改：

### 旧语法匿名委托不转换为表达式树

与本机编译器一样，Roslyn C# 不会将旧式匿名方法表达式转换为表达式树。

```cs
using System;
using System.Linq.Expressions;
class Program
{
    static void Main()
    {
        Foo(delegate { }); // 没有错误；选择非表达式版本。
    }
    static void Foo(Action a) { }
    static void Foo(Expression<Action> a) { }
}
```

### 枚举运算符 -

本机编译器在枚举类型的减法运算符方面有一些奇怪的行为。这些不太可能影响真实代码，但我们重现它们以确保不会通过改变其行为来破坏程序。

旧编译器实现了内置运算符：

```cs
	E operator-(I, E)
```

其中 `E` 是底层类型为 `I` 的枚举类型。

对规范的精确阅读表明，对于底层类型不是 int 的枚举变量 `e`，表达式 `e - 0` 是有歧义的（0 可以转换为枚举类型或底层类型，但两者都不是精确匹配）。我们与旧编译器一样解析该歧义。

你可以使用编译器标志 /features:strict 关闭这些兼容性功能。

### 多接口继承的平局规则

旧编译器在存在可选参数和 `params` 参数时实现了（语言规范中没有的）特殊重载解析规则，Roslyn 对规范更严格的解释（现已修复）阻止了某些程序编译。

具体来说，如果两个具有相同名称的扩展方法在不同的接口上实现，并且这两个接口都由一个单一的接口扩展（例如称为 `IFinalInterface`），使得该接口包含两个同名的扩展方法，就会出现问题。

第一个扩展方法以 `params int[]` 作为参数：

```cs
public static int Properties(this IFirstInterface source, params int[] x) 
{ 
    return 0; 
}
```
 
第二个扩展方法有一个额外的可选参数：

```cs
public static bool Properties(this ISecondInterface source, int x = 0, params int[] y) 
{ 
return true; 
} 
```

现在，以下调用会导致歧义错误：

```cs 
var x = default(IFinalInterface); 
var properties = x.Properties();
```

有关更多详情，请参见 #2298，有关使 Roslyn 能够编译此代码的实现和测试，请参见 #2305。

### 未使用 param-array 参数的平局规则

旧编译器在存在未使用的 param-array 参数时实现了（语言规范中没有的）特殊重载解析规则，Roslyn 对规范更严格的解释（现已修复）阻止了某些程序编译。

具体来说，如果两个方法具有相同的参数类型（除了 param-array 类型之外），并且尝试不为 param-array 参数提供任何参数来调用它们，就会出现问题。

```
public class Test
{
    public Test(int a, params string[] p) { }
    public Test(int a, params List<string>[] p) { }
}
```

现在，以下调用会导致歧义错误：

```
var x = new Test(10);
```

旧编译器调用第二个重载（```Test(int a, params List<string>[] p)```）。

有关更多详情，请参见 https://github.com/dotnet/roslyn/issues/4458，有关使 Roslyn 能够编译此代码的实现和测试，请参见 https://github.com/dotnet/roslyn/issues/4761。

### 编译器不会提升不对称的 `operator==` 和 `operator!=`

C# 编译器（所有版本）不会提升用户定义的相等运算符：

``` c#
public static bool operator ==(Type1 x, Type2 y)
public static bool operator !=(Type1 x, Type2 y)
```

除非 `Type1` 和 `Type2` 是相同的类型。规范要求在值类型上定义的所有用户定义的相等运算符也以提升形式考虑，因此从技术上讲这违反了规范。我们认为修复这种长期以来的行为可能会破坏某些程序，因此不期望改变它。

有关更多详情，请参见 https://github.com/dotnet/roslyn/issues/13686。
