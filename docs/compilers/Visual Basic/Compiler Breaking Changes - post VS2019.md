## 本文档列出了 *Visual Studio 2019 Update 1* 及以后版本相对于 *Visual Studio 2019* 在 Visual Basic Roslyn 中的已知重大更改。

1. https://github.com/dotnet/roslyn/issues/38305 

编译器以前在将产生 Boolean? 的内置比较运算符用作逻辑短路运算符作为布尔表达式的操作数时，会生成不正确的代码。
例如，对于表达式：
```
    GetBool3() = True AndAlso GetBool2()
```
    以及函数：
```
    Function GetBool2() As Boolean
        System.Console.WriteLine("GetBool2")
        Return True
    End Function
    Shared Function GetBool3() As Boolean?
         Return Nothing
    End Function
```

预期 GetBool2 函数将被调用。这也是在布尔表达式上下文之外的预期行为，但在布尔表达式上下文中，GetBool2 函数没有被调用。
编译器现在生成遵循语言语义的代码，并为上述表达式调用 GetBool2 函数。
