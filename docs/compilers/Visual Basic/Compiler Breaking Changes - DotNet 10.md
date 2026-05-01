# .NET 9.0.100 至 .NET 10.0.100 之间 Visual Basic Roslyn 的重大更改

本文档列出了在 .NET 9 正式发布（.NET SDK 版本 9.0.100）之后到 .NET 10 正式发布（.NET SDK 版本 10.0.100）之间，Visual Basic Roslyn 中已知的重大更改。

## 在处置过程中将枚举器对象的状态设置为"之后"

***在 Visual Studio 2022 版本 17.13 中引入***

枚举器的状态机错误地允许在枚举器被处置后恢复执行。  
现在，对已处置的枚举器调用 `MoveNext()` 会正确返回 `false`，而不会执行任何更多的用户代码。

```vb
Imports System
Imports System.Collections.Generic

Module Program
    Sub Main()
        Dim enumerator = C.GetEnumerator()

        Console.Write(enumerator.MoveNext()) ' 打印 True
        Console.Write(enumerator.Current)    ' 打印 1

        enumerator.Dispose()

        Console.Write(enumerator.MoveNext()) ' 现在打印 False
    End Sub
End Module

Class C
    Public Shared Iterator Function GetEnumerator() As IEnumerator(Of Integer)
        Yield 1
        Console.Write("处置后不执行")
        Yield 2
    End Function
End Class
```

## `Microsoft.CodeAnalysis.EmbeddedAttribute` 在声明时进行验证

***在 Visual Studio 2022 版本 17.13 中引入***

编译器现在在源代码中声明 `Microsoft.CodeAnalysis.EmbeddedAttribute` 时验证其形状。以前，编译器允许用户以任何形状定义此属性。现在我们验证以下条件：

1. 必须是 Friend（友元）
2. 必须是 Class（类）
3. 必须是 NotInheritable（不可继承）
4. 不得是 Module（模块）
5. 必须有公共无参数构造函数
6. 必须继承自 System.Attribute
7. 必须允许用于任何类型声明（Class、Structure、Interface、Enum 或 Delegate）

```vb
Namespace Microsoft.CodeAnalysis

    ' 以前允许。现在，BC37335
    Public Class EmbeddedAttribute
        Inherits Attribute
    End Class
End Namespace
```
