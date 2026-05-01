# 目标框架策略

## 层次结构

roslyn 仓库为许多不同的产品生成组件，这些产品对我们施加了不同的发布和 TFM 约束。我们的一些依赖项摘要如下：

- 构建工具：要求我们在 `net472` 上发布编译器
- .NET SDK：要求我们在当前服务目标框架（目前为 `net10.0`）上发布编译器
- 仓库源构建：要求我们在工作区及以下层发布 `$(NetCurrent)` 和 `$(NetPrevious)`（目前分别为 `net10.0` 和 `net9.0`）。这是因为仓库源构建的输出是其他仓库源构建的输入，而那些构建可能针对 `$(NetCurrent)` 或 `$(NetPrevious)`。
- 完整源构建：要求我们发布 `$(NetCurrent)`
- Visual Studio：要求我们为基础 IDE 组件发布 `net472`，为私有运行时组件发布 `$(NetVisualStudio)`（目前为 `net8.0`）。
- Visual Studio Code：期望我们针对与 DevKit 相同的运行时（目前为 `net10.0`）发布，以避免两次运行时下载。
- MSBuildWorkspace：要求我们发布一个必须在最低支持的 SDK（目前为 `net8.0`）上可用的进程

对我们来说，取所有 TFM 的并集并将每个项目多目标化是不合理的。这将为任何构建操作添加数百次编译，从而对开发者吞吐量产生负面影响。相反，我们尝试在需要时使用 TFM。这使我们的构建更小，但增加了一些复杂性，因为我们最终为各层的二进制文件发布混合 TFM。

## 选择正确的 TargetFramework

仓库中的项目应根据以下规则在 `<TargetFramework(s)>` 中包含以下值：

1. `$(NetRoslynSourceBuild)`：需要成为源构建一部分的代码。此属性将根据代码是在源构建上下文还是官方构建中构建而变化。
   a. 在官方构建中，这将包含 `$(NetVSShared)` 的 TFM
   b. 在源构建中，这将包含 `$(NetRoslyn)`
2. `$(NetVS)`：需要在 Visual Studio 私有运行时上执行的代码。
3. `$(NetVSCode)`：需要在 DevKit 主机中执行的代码
4. `$(NetVSShared)`：需要在 Visual Studio 和 VS Code 中执行但不需要被源构建的代码。
5. `$(NetRoslyn)`：需要在 .NET 上执行但没有任何特定产品部署要求的代码。例如，被我们的基础设施使用的工具、编译器单元测试等。此属性还控制编译器针对的哪些框架在工具集包中发布。此值在源构建中可能会改变。
6. `$(NetRoslynAll)`：通常是测试工具，需要为我们支持的所有 .NET 运行时构建的代码。
7. `$(NetRoslynWindowsTests)`：使用 `$(NetRoslyn)` 但只能在 Windows 上运行的测试项目。由于 `-windows` OS 特定的 TFM 后缀告诉 .NET SDK 和测试运行器在非 Windows 平台上跳过它们，这些测试在 Unix/Linux 上运行时将自动被排除。
8. `$(NetRoslynBuildHostNetCoreVersion)`：MSBuildWorkspace 使用的 .NET Core BuildHost 进程的目标。
9. `$(NetRoslynNext)`：需要在下一个 .NET Core 版本上运行的代码。这在过渡到新 .NET Core 版本时使用，我们需要向前推进但不想在构建文件中硬编码 .NET Core TFM。

属性 `$(NetCurrent)`、`$(NetPrevious)` 和 `$(NetMinimum)` 不在我们的项目文件中使用，因为它们的变化方式使我们难以维护正确的产品部署。我们的产品在 VS 和 VS Code 上发布，这不被 arcade 的 `$(Net...)` 宏捕获。此外，随着 arcade 属性的变化，很容易导致 `<TargetFrameworks>` 设置中出现重复条目。相反，我们的仓库使用上述值，当在源构建或 VMR 内部时，我们的属性用 arcade 属性初始化。

**不要**在项目文件中硬编码 .NET Core TFM。应使用上述属性，因为这让我们可以集中管理它们并构造属性以避免重复。可以硬编码其他 TFM，如 `netstandard2.0` 或 `net472`，因为这些不预期会更改。

**不要**在项目文件中使用 `$(NetCurrent)` 或 `$(NetPrevious)`。这些只应在 `TargetFrameworks.props` 内部使用，以在某些配置中初始化上述值。

## 跨目标框架要求一致的 API

确保我们的发布 API 在各目标框架中保持一致的 API 表面积非常重要。无论 API 是 `public` 还是 `internal`，这都是适用的。

`public` 的原因是标准设计模式。`internal` 的原因是以下问题的组合：

- 我们的仓库使用 `InternalsVisibleTo`，允许其他程序集直接引用 `internal` 成员的签名。
- 我们的仓库发布混合目标框架。通常工作区及以下层会发布比上层更新的 TFM。编译器必须为源构建发布更新的 TFM，而 IDE 受 Visual Studio 私有运行时约束，因此采用更新 TFM 的速度较慢。
- 我们的仓库投资于 polyfill API，使在同一项目中针对多个 TFM 编译成为无缝体验。

综合来看，这意味着在许多情况下，我们的 `internal` 表面积在二进制兼容性方面实际上是 `public`。例如，消费项目可能从工作区层获得 `net7.0` 二进制文件，从 IDE 层获得 `net6.0` 二进制文件。由于这些二进制文件之间存在 `InternalsVisibleTo`，`internal` API 表面积实际上是 `public` 的。这要求我们有一致的策略来实现跨 TFM 组合的二进制兼容性。

## 过渡到新 .NET SDK

随着 .NET 团队接近发布新的 .NET SDK，Roslyn 团队将开始在我们的构建中使用该 SDK 的预览版本。这通常会由于运行时中的底层行为变化而在我们的 CI 系统中导致测试失败。这些失败通常不会在 Visual Studio 中运行时出现，因为测试运行器的运行时选择方式不同。

为了确保我们有简单的开发者环境，此类项目应移动到 `$(NetRoslynNext)` 目标框架。这确保在本地运行测试时加载新的运行时。

当 .NET SDK RTM 且 Roslyn 采用它时，所有 `$(NetRoslynNext)` 的出现将简单地移动到 `$(NetRoslyn)`。

**不要**在同一项目中同时包含 `$(NetRoslyn)` 和 `$(NetRoslynNext)`，除非有非常具体的原因说明两个测试都在增加价值。最常见的情况是运行时改变了行为，我们只需要更新基准来匹配它。添加额外的 TFM 只会增加测试时间，而收益很小。

## 更新 TFM 的检查清单（每年一次）

- 更新 `TargetFrameworks.props`。
  - 确保我们有正确的 VS / VSCode TFM（通常不是最新的 TFM）。
- 可能需要更新 `eng\Versions.props` 中的 `MicrosoftNetCompilersToolsetVersion` 以使用最新的编译器功能。
- 将项目文件中的 `$(NetRoslynNext)` 引用更改为 `$(NetRoslyn)`。
- 更新代码/脚本/管道中的 TFM（搜索旧的 `netX.0` 并替换为新的 `netY.0`）。
- 将对应于不再使用的 TFM 的 `#if NET7_0_OR_GREATER`（在本例中为 `net7.0`）替换为最低使用的 TFM（例如，`net8.0`：`#if NET8_0_OR_GREATER`），同样地替换取反变体 `#if !NET7_0_OR_GREATER` 为 `#if !NET8_0_OR_GREATER`。
- 检查 CI 中是否仍然运行相同数量的测试（它们没有被 TFM 无意中过滤掉）。
- 尝试官方构建，VS 插入。
