# 与 C# 标准的偏差

本文档按标准章节组织，列出了 Roslyn 与 C# 标准之间已知的不一致之处。

## 转换

### 从零的隐式枚举转换

来自 [§10.2.4](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/conversions.md#1024-implicit-enumeration-conversions)：

> 隐式枚举转换允许将任何整数类型且值为零的 *constant_expression*（[§12.25](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/expressions.md#1225-constant-expressions)）转换为任何 *enum_type* 以及任何底层类型为 *enum_type* 的 *nullable_value_type*。

Roslyn 也从 `float`、`double` 和 `decimal` 类型的常量表达式执行隐式枚举转换：

```csharp
enum SampleEnum
{
    Zero = 0,
    One = 1
}

class EnumConversionTest
{
    const float ConstFloat = 0f;
    const double ConstDouble = 0d;
    const decimal ConstDecimal = 0m;

    static void PermittedConversions()
    {
        SampleEnum floatToEnum = ConstFloat;
        SampleEnum doubleToEnum = ConstDouble;
        SampleEnum decimalToEnum = ConstDecimal;
    }
}
```

从 `bool` 类型、其他枚举或引用类型的常量表达式进行转换（正确地）*不*被允许。

## 成员查找

来自 [§12.5.1](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/expressions.md#125-member-lookup)：

> - 最后，删除隐藏的成员后，确定查找的结果：
>   - 如果集合只包含一个非方法成员，则该成员是查找的结果。
>   - 否则，如果集合只包含方法，则这组方法是查找的结果。
>   - 否则，查找是模糊的，发生绑定时错误。

Roslyn 改为实现了对方法优于非方法符号的偏好：

```csharp
var x = I.M; // 绑定到 I1.M（方法）
x();

System.Action y = I.M; // 绑定到 I1.M（方法）

interface I1 { static void M() { } }
interface I2 { static int M => 0;   }
interface I3 { static int M = 0;   }
interface I : I1, I2, I3 { }
```

```csharp
I i = null;
var x = i.M; // 绑定到 I1.M（方法）
x();

System.Action y = i.M; // 绑定到 I1.M（方法）

interface I1 { void M() { } }
interface I2 { int M => 0;   }
interface I : I1, I2 { }
```

## 关于已知类型/成员的假设

编译器可以自由地对已知类型/成员的形状和行为做出假设。
它可能不会检查意外的约束、`Obsolete` 属性或 `UnmanagedCallersOnly` 属性。
它可能会基于类型/成员行为良好的预期执行某些优化。
注意：编译器应在缺少已知类型/成员时保持健壮性。

## 接口分部方法

接口分部方法隐式为非虚，与非分部接口方法和其他接口分部成员类型不同，
参见[相关重大更改](./Compiler%20Breaking%20Changes%20-%20DotNet%2010.md#partial-properties-and-events-are-now-implicitly-virtual-and-public)
以及 [LDM 2025-04-07](https://github.com/dotnet/csharplang/blob/main/meetings/2025/LDM-2025-04-07.md#breaking-change-discussion-making-partial-members-in-interfaces-virtual-andor-public)。
