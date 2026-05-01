已用程序集引用
=========================

*已用程序集引用*功能在 `Compilation` 上提供了 `GetUsedAssemblyReferences` API，用于获取编译中被认为已使用的元数据程序集引用集合。例如，如果在编译的源代码中引用了某个被引用程序集中声明的类型，则该引用被视为已使用。以此类推。

更多信息请参见 https://github.com/dotnet/roslyn/issues/37768。
