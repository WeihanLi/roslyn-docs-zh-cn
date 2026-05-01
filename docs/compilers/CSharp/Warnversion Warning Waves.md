# /warn 警告"波次"

C# 编译器标志 `/warn` 控制可选警告。
当我们引入可能在现有代码上报告的新警告时，
我们通过一个选择加入系统来实现，以便程序员在采取行动启用之前不会看到新警告。
为此，我们有编译器标志"`/warn:n`"，其中 `n` 是一个整数。

使用命令行编译器时的默认警告级别为 `4`。如果你希望编译器产生所有适用的警告，可以指定 `/warn:9999`。

在典型项目中，此设置由 `AnalysisLevel` 属性控制，该属性确定 `WarningLevel` 属性（传递给 `Csc` 任务）。
有关 `AnalysisLevel` 的更多信息，请参见 https://devblogs.microsoft.com/dotnet/automatically-find-latent-bugs-in-your-code-with-net-5/

## 警告级别 11

随 .NET 11（C# 15 编译器）一起发布的编译器包含以下仅在 `/warn:11` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS9377 | [在当前规则下，'unsafe' 修饰符在此处没有任何效果](https://github.com/dotnet/csharplang/issues/9704) |

## 警告级别 10

随 .NET 10（C# 14 编译器）一起发布的编译器包含以下仅在 `/warn:10` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS9265 | [字段永远不会被 ref 赋值，并且始终具有其默认值（null 引用）](https://github.com/dotnet/roslyn/issues/75315) |

## 警告级别 8

随 .NET 8（C# 12 编译器）一起发布的编译器包含以下仅在 `/warn:8` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS9123 | [在 async 方法中获取局部变量或参数的地址可能会产生 GC 漏洞](https://github.com/dotnet/roslyn/issues/63100) |
| EnableGenerateDocumentationFile | [用于在构建时强制执行 IDE0005 的辅助诊断](https://github.com/dotnet/roslyn/issues/70460) |

## 警告级别 7

随 .NET 7（C# 11 编译器）一起发布的编译器包含以下仅在 `/warn:7` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS8981 | [仅包含小写 ASCII 字符的类型名称可能会成为语言的保留名称](https://github.com/dotnet/roslyn/issues/56653) |

## 警告级别 6

随 .NET 6（C# 10 编译器）一起发布的编译器包含以下仅在 `/warn:6` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS8826 | [分部方法声明有签名差异](https://github.com/dotnet/roslyn/issues/47838) |

## 警告级别 5

随 .NET 5（C# 9 编译器）一起发布的编译器包含以下仅在 `/warn:5` 或更高级别下报告的警告。

| 警告 ID | 描述 |
|------------|-------------|
| CS7023 | [在 'is' 或 'as' 表达式中使用了静态类型](https://github.com/dotnet/roslyn/issues/30198) |
| CS8073 | [将结构与 null 比较时，表达式始终为 true（或 false）](https://github.com/dotnet/roslyn/issues/45744) |
| CS8848 | [诊断查询表达式中的优先级错误](https://github.com/dotnet/roslyn/issues/30231) |
| CS8880 | [结构构造函数未赋值自动属性（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8881 | [结构构造函数未赋值字段（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8882 | [out 参数未赋值（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8883 | [在结构构造函数中赋值所有字段之前使用了自动属性（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8884 | [在结构构造函数中赋值之前使用了字段（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8885 | [结构构造函数在赋值所有字段之前读取了 'this'（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8886 | [out 参数在赋值之前被使用（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8887 | [局部变量在赋值之前被使用（具有私有字段的导入结构类型）](https://github.com/dotnet/roslyn/issues/30194) |
| CS8892 | [多个入口点](https://github.com/dotnet/roslyn/issues/46831) |
| CS8897 | [在接口类型的方法中使用了静态类作为参数类型](https://github.com/dotnet/roslyn/issues/38256) |
| CS8898 | [在接口类型的方法中使用了静态类作为返回类型](https://github.com/dotnet/roslyn/issues/38256) |
