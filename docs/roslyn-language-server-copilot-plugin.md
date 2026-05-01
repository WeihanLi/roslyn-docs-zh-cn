# Roslyn 语言服务器 - Copilot 插件

`roslyn-language-server` 是一个 .NET 工具，通过[语言服务器协议（LSP）](https://microsoft.github.io/language-server-protocol/)向 AI 编码助手提供 C# 语言智能（跳转到定义、查找引用、诊断等）。它通过 [dotnet/skills](https://github.com/dotnet/skills) 市场以插件形式分发，支持在 GitHub Copilot CLI 等工具中自动安装和使用。

## 快速上手

对于本仓库，等效的 Copilot CLI LSP 配置已签入 `.github/lsp.json`，且 `.vscode/settings.json` 将服务器指向 `Roslyn.slnx`。这意味着在 Copilot CLI 中打开本仓库即可自动完成 C# LSP 配置。

### 安装插件

在你的 AI 助手（Copilot CLI 等）中，添加市场并安装 `dotnet` 插件：

```
/plugin marketplace add dotnet/skills
/plugin install dotnet@dotnet-agent-skills
```

重启助手以加载插件。安装完成后，助手将通过 LSP 自动获得对 `.cs` 文件的 C# 语言智能支持。

### 功能说明

激活插件后，助手将获得以下由 LSP 驱动的 C# 代码能力：

- **跳转到定义** — 导航到符号定义处
- **查找引用** — 找到符号的所有使用位置
- **悬停提示** — 获取类型信息和文档
- **诊断** — 查看编译器错误和警告
- **文档符号** — 列出文件中的所有符号
- **更多功能** — Roslyn 语言服务器支持的任何 LSP 功能

## 前提条件

- 必须安装 **.NET 10 SDK** 并将其添加到 `PATH`。你的项目仍可使用旧版受支持的 SDK。

## 自动项目加载

语言服务器按以下策略（按顺序评估）自动发现并加载项目：

### 1. VS Code 设置（`dotnet.defaultSolution`）

如果工作区文件夹中存在 `.vscode/settings.json` 文件，服务器将读取 `dotnet.defaultSolution` 设置：

```jsonc
// .vscode/settings.json
{
  "dotnet.defaultSolution": "src/MyApp.sln"
}
```

- 支持指向 `.sln` 或 `.slnx` 文件的**相对路径或绝对路径**。
- 设置为 `"disable"` 可阻止服务器自动加载任何解决方案或项目：
  ```jsonc
  {
    "dotnet.defaultSolution": "disable"
  }
  ```

### 2. 根目录下的单个解决方案文件

如果工作区文件夹根目录下**恰好有一个** `.sln` 或 `.slnx` 文件，服务器将自动加载该解决方案。

### 3. 单个项目发现

作为后备方案，服务器会递归发现工作区文件夹中的所有 `.csproj` 文件并逐一加载。

## 故障排除

### 验证 LSP 配置是否被识别

- 在 Copilot CLI 中，运行 `/lsp show` 应显示为 C# 配置的服务器：
```
  Plugin-configured servers:
    • csharp: (.cs) [from dotnet]
```

### 查看 LSP 日志

Copilot CLI 将 LSP 服务器日志写入用户主目录下的 `.copilot/logs/` 目录。检查语言服务器输出的方法：

- **macOS / Linux：** `~/.copilot/logs/`
- **Windows：** `%USERPROFILE%\.copilot\logs\`

查找与 `csharp` 或 `roslyn-language-server` 相关的日志文件。这些日志包含服务器的启动输出、项目加载进度和遇到的错误，是语言服务器行为异常时的首要排查位置。

### 找不到工具

- 确保 `dotnet` 已添加到 `PATH`。
- 检查是否存在限制包源的 `nuget.config`（仓库级、用户级或机器级）；`dnx` 使用 NuGet 源来解析工具。
- 验证是否启用了 `nuget.org`（或其他镜像 `roslyn-language-server` 的源），例如使用 `dotnet nuget list source`。
- 尝试手动安装工具：`dotnet tool install -g roslyn-language-server --prerelease`。

### 项目加载问题

如果 LSP 日志显示项目加载失败：

- 运行 `dotnet --version` 验证是否安装了兼容的 .NET SDK。
- 在使用语言服务器之前，使用 `dotnet build` 确认项目能够成功构建。
- 对于包含多个解决方案的大型仓库，在 `.vscode/settings.json` 中配置 `dotnet.defaultSolution` 以指定要加载的解决方案。

### 性能

- 加载包含大量项目的解决方案可能需要一些时间。服务器在加载过程中会向客户端报告进度。
- 对于大型仓库，建议通过 `dotnet.defaultSolution` 加载特定解决方案，而不是依赖单个项目发现（后者可能会加载不需要的测试项目和其他项目）。

## 相关链接

- [dotnet/skills 仓库](https://github.com/dotnet/skills) — .NET 代理技能的插件市场

## 工作原理

该插件通过 [dotnet/skills](https://github.com/dotnet/skills) 仓库中的 [`plugins/dotnet/lsp.json`](https://github.com/dotnet/skills/blob/main/plugins/dotnet/lsp.json) 进行配置：

当助手打开包含 `.cs` 文件的工作区时，将：

1. 使用 `dotnet dnx` **按需安装并运行** [`roslyn-language-server`](https://www.nuget.org/packages/roslyn-language-server) .NET 工具（该命令会自动下载并缓存工具；`--yes` 跳过确认，`--prerelease` 允许预发布版本）。
2. 使用语言服务器协议通过 **stdio**（`--stdio`）进行通信。
3. **自动发现并加载项目**（`--autoLoadProjects`），使助手立即获得对代码库的完整语义理解。

### 命令行选项

| 选项 | 说明 |
|------|------|
| `--stdio` | 使用 stdio 进行 LSP 通信（大多数代理集成所必需） |
| `--autoLoadProjects` | 自动发现并加载工作区文件夹中的项目 |
| `--logLevel <level>` | 最低日志详细程度（默认：`Information`） |
| `--debug` | 启动时打开调试器 |
