### VB 嵌入式运行时从编译中继承溢出检查

参见 https://github.com/dotnet/roslyn/issues/6941。某些 VB 运行时方法被指定为在转换的值溢出目标类型时抛出 `OverflowException`。当使用嵌入的运行时（`/vbruntime*`）编译 VB 项目时，编译器会将必要的 VB 运行时辅助方法包含到生成的程序集中。这些运行时辅助方法从嵌入它们的 VB.NET 项目中继承溢出检查行为。因此，如果你既嵌入运行时又禁用了溢出检查（`/removeintchecks+`），你将不会从运行时辅助方法中获得指定的异常。虽然从技术上讲这是一个 bug，但它长期以来都是 VB.NET 的行为，我们发现客户会因修复它而受损，因此我们不期望改变这种行为。

``` vb
Sub Main()
    Dim s As SByte = -128
    Dim o As Object = s
    Dim b = CByte(o) ' 应该抛出 OverflowException，但如果你使用 /vbruntime* /removeintchecks+ 编译则不会
    Console.WriteLine(b)
End Sub
```
