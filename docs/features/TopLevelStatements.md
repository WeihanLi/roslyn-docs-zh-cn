顶级语句
=========================

允许一系列*语句*直接出现在*编译单元*（即源文件）的*命名空间成员声明*之前。

其语义为：如果存在此类*语句*序列，则会生成以下类型声明（实际类型名称和方法名称除外）：

``` c#
static class Program
{
    static async Task Main()
    {
        // statements
    }
}
```

提案：https://github.com/dotnet/csharplang/blob/main/proposals/top-level-statements.md
未解决问题和待办事项在 https://github.com/dotnet/roslyn/issues/41704 中跟踪。
测试计划：https://github.com/dotnet/roslyn/issues/43563。
另请参见 https://github.com/dotnet/csharplang/issues/3117。
