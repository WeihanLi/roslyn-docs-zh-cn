# 外部访问策略

## API 跟踪

我们使用 roslyn 分析器跟踪我们公开的 API，如果 API 被删除，构建将失败。与我们的公共 API 不同，我们允许在这里进行破坏性更改，但这些通常会导致插入失败。任何现有 API 的删除（无论 API 是否完全删除，还是参数/返回值更改）都需要获得当前基础设施轮换的批准，并且可能需要在合并之前进行测试插入以确保 VS 不会被破坏，或者与受影响的合作伙伴团队协调插入。

因为 EA 项目的"发布"在每次检入更改时发生，所以我们不费心更新这些项目的 `InternalAPI.Shipped.txt` 文件。

每个 EA 项目都有一个 `Internal` 命名空间，被视为 Roslyn 实现细节。例如，OmniSharp 命名空间是 `Microsoft.CodeAnalysis.ExternalAccess.OmniSharp.Internal`。我们不跟踪这些命名空间下的 API，依赖项目有责任不引用这些命名空间中的任何内容。并非每个 EA 项目都实际使用此命名空间。如果当前未使用其 `Internal` 命名空间的 EA 项目想要开始使用，请修改 `/src/Tools/ExternalAccess/.editorconfig` 以将受影响文件的 `dotnet_public_api_analyzer.skip_namespaces` 键设置为该命名空间。可以使用 `,` 分隔符包含多个命名空间。

## OmniSharp

当需要更改 ExternalAccess.OmniSharp 或 ExternalAccess.OmniSharp.CSharp 包中的 API 时，请 ping @333fred、@JoeRobich、@filipw 或 @david-driscoll 作为提前通知。允许破坏性更改，但请等待确认和后续问题以确保我们不会完全破坏 OmniSharp 场景。
