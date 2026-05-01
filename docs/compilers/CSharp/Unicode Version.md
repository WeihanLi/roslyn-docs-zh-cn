# C# 6 中的 Unicode 版本变更

Roslyn 编译器依赖底层平台实现 Unicode 行为。实际上，这意味着新编译器将反映 Unicode 标准中的变更。

例如，Unicode 片假名中间点"・"（U+30FB）在 C# 6 中不再适用于标识符。
其 Unicode 类在 Unicode 5.1 或更旧版本中为 Pc（标点符号，连接符），但在 Unicode 6.0 中更改为 Po（标点符号，其他）。

另请参见 https://github.com/ufcpp/UfcppSample/blob/master/BreakingChanges/VS2015_CS6/KatakanaMiddleDot.cs
