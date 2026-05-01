# Codespaces

本仓库提供了用于完整 Roslyn 仓库工作的 GitHub Codespaces 配置。

## 可用配置

| 配置 | 默认解决方案 | 预期用途 | 最低机器配置 |
| --- | --- | --- | --- |
| `Roslyn (.NET 10)` | `Roslyn.slnx` | 完整仓库工作 | 8 核，16 GB RAM，32 GB 存储 |

该配置使用 `.devcontainer/Dockerfile`，从 .NET 10 SDK 镜像开始，然后安装由 `global.json` 固定的确切 SDK 版本。

## 生命周期选择

仓库在 `updateContentCommand` 中使用 `.devcontainer/restore-workspace.sh <solution>`。

这一放置是有意为之的：

- `updateContentCommand` 在 Codespaces 预构建创建期间运行。
- `postCreateCommand` 在预构建创建期间不运行。
- 将恢复操作保留在 `updateContentCommand` 中允许 GitHub Codespaces 缓存耗时的首次运行恢复步骤。

该配置还通过开发容器功能安装 `gh` 和 `pwsh`，使容器更贴近常见的 Roslyn 命令行工作流。

## 推荐的仓库级 GitHub 设置

这些设置位于仓库的 **Settings > Codespaces** 页面，而非仓库本身。

1. 使用 `.devcontainer/devcontainer.json` 为默认分支启用预构建。
2. 从**配置更改时**触发器开始，以控制存储和 Actions 的使用量。
3. 将保留的预构建版本数设为 `2`，除非您有明确的更长回滚窗口需求。
4. 将预构建区域限制为您的团队实际使用的区域。

## 验证清单

更改 Codespaces 设置或开发容器配置后：

1. 从 `.devcontainer/devcontainer.json` 创建一个全新的 codespace。
2. 确认预期的默认解决方案在 VS Code 中加载。
3. 确认 `gh --version` 和 `pwsh --version` 在终端中正常工作。
4. 确认所选解决方案的恢复步骤成功完成。
