# VS2017（VB 15）之后的更改

- https://github.com/dotnet/csharplang/issues/415
在 VB 15 中，元组字面量中的元素可以显式命名，但在 VB 15.3 中，没有显式命名的元素将获得推断名称。这使用与匿名类型中未显式命名的成员相同的规则。
例如，`Dim t = (a, b.c, Me.d)` 将生成元素名称为 "a"、"c" 和 "d" 的元组。因此，对元组成员的调用可能会得到与 VB 15 不同的结果。
考虑 `a` 的类型为 `System.Func(Of Boolean)` 的情况，写 `Dim local = t.a()`。现在将找到元组的第一个元素并调用它，而以前只能意味着"调用名为 'a' 的扩展方法"。

- https://github.com/dotnet/roslyn/issues/20873 在 Roslyn 2.3 中，`EmitOptions` 构造函数的 `includePrivateMembers` 参数被更改为使用 `true` 作为默认值。这是一个二进制兼容性中断。因此，使用此 API 的客户端可能需要重新编译，以获取新的默认值。将包含一个缓解措施（在尝试发出完整程序集时忽略旧的默认值）。

- Visual Studio 2017 版本 15.8：https://github.com/dotnet/roslyn/pull/27461 `LanguageVersionFacts.TryParse` 方法不再是扩展方法。
