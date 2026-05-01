# 编译器测试中的本地化

编译器测试的结构设计使其可以在使用任何语言区域设置的机器上运行。这既是确保编译器不存在本地化错误的工具，也是确保来自任何语言区域的开发者都能参与项目贡献的手段。构建和执行测试的能力不应仅限于讲英语的开发者。

我们的基础设施在 `es-ES` 机器上针对 .NET Core 和 .NET Framework 运行编译器测试套件。这有助于确保编译器测试可以在非英语机器上编写和执行，不会出现问题。

## 工作原理

为确保测试能在任何语言区域设置下运行，我们遵循以下通用方法：

1. 所有预期基准值均以不变文化（invariant culture）表示。这意味着例如 `decimal` 值使用 `.` 而非 `,` 作为分隔符。
2. 所有生成的基准值（通常通过 `CompileAndVerify` 内部的 `Console.Writeline`）必须以不变文化格式生成输出。
3. 诊断测试使用 `VerifyDiagnostics` 帮助方法，并使用 `en-US` 值作为消息。`VerifyDiagnostics` 的实现将以 `en-US` 文化请求诊断消息。

这有助于确保编译器不存在本地化产品问题，无论系统当前的文化设置如何，都强制使用一致的文化。

开发者在编写测试时遇到的最大摩擦来源是向 `Console.WriteLine` 传递具有依赖语言区域表示的值，例如 `double` 和 `decimal` 值。

```csharp
decimal d = 1.2;
Console.WriteLine(d);
```

上述代码在 `en-US` 机器上打印 `1.2`，但在 `es-ES` 机器上打印 `1,2`。要解决此问题，需要显式使用不变文化格式化该值。

```csharp
using System.Globalization;
...

decimal d = 1.2;
Console.WriteLine(d.ToString(CultureInfo.InvariantCulture));
```

这将始终打印 `1.2`。

## 例外情况

在某些情况下很难避免依赖语言区域的值，包括：

1. 当诊断消息包含来自抛出异常的消息时。
2. 当诊断消息包含来自底层操作系统的消息时。
3. 当消息由 `msbuild` 等工具生成时。

在这些情况下，消息可能依赖于语言区域。具有这些值的测试应标记为 `[ConditionalFact(typeof(IsEnglishLocal))]`，使其仅在 `en-US` 机器上运行。

## 基础设施

### Windows

我们基础设施中使用的西班牙语 Windows 机器以 `CurrentCulture` 设置为 `es-ES`，但 `CurrentUICulture` 设置为 `en-US` 的方式执行。此配置意味着字符串格式化大致使用 `es-ES`，但资源查找使用 `en-US`。这种设置不能完全测试我们的代码库，因此当两者不同时，我们的测试基础设施将强制将 `CurrentUICulture` 设置为 `CurrentCulture`。

这在 `TestBase` 构造函数中完成，所有编译器测试类都继承自该构造函数。当 `CurrentUICulture` 不等于 `CurrentCulture` 时，构造函数保存原始的 `CurrentUICulture` 并将 `CurrentUICulture` 设置为 `CurrentCulture`。原始值在 `Dispose` 中恢复，因此更改的范围限定为每个测试实例的生命周期。这确保在配置了 `CurrentCulture = es-ES` 和 `CurrentUICulture = en-US` 的 Windows 机器上，资源字符串查找也以 `es-ES` 进行，从而充分测试本地化路径。

### Linux

我们基础设施中使用的西班牙语 Linux 机器在 Ubuntu 上运行，并将 `LC_ALL` 环境变量设置为 `es_ES.UTF-8`。在 Linux 上运行的 .NET 中，`LC_ALL` 同时决定 `CurrentCulture` 和 `CurrentUICulture`。这意味着与 Windows 配置不同，字符串格式化和资源查找都将使用 `es-ES`。由于两个文化值已经相同，测试基础设施不需要在这些机器上强制将 `CurrentUICulture` 设置为 `CurrentCulture`。
