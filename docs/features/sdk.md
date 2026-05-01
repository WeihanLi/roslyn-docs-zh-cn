将 dotnet/roslyn 迁移到新 SDK
=============

# 概述

本文档跟踪我们将 Roslyn 迁移到新 SDK 和项目系统的进度。此次初步推进的明确目标是从我们的代码库中移除 project.json 的使用。在此过程中，我们的许多项目将迁移到新 SDK，但不是全部。一旦此变更合并，我们将进行进一步清理，将所有内容迁移到新 SDK。

我们仓库中需要转换的项目分为以下几类：

- PCL：将转换为新 SDK 和项目系统
- Desktop 4.6：包使用将改为 PackageReference
- Desktop 2.0：包使用将改为 PackageReference

我们的项目转换几乎完全通过 [ConvertPackageRef 工具](https://github.com/jaredpar/ConvertPackageRef) 自动化完成。此自动化操作始终作为标题为 "Generated code" 的单个提交检入，随着工具的改进，会频繁通过 rebase 替换。

# 分支状态

所有故意禁用的功能都用 "DO NOT MERGE" 注释标记。这些条目将在合并前移除并修复。

## 正常运行

- 开发者工作流：Restore.cmd、Build.cmd、Test.cmd
- 在 Visual Studio 中编辑

## Jenkins 测试阶段

以下 Jenkins 阶段被认为是功能性的，且每次合并都必须通过：

- x86 / amd64 上的调试/发布单元测试
- 构建正确性
- Microbuild
- CoreClr
- Linux
- 集成测试

## 重大事项

以下是已损坏且需要一些工作的重大事项：

- RepoUtil：需要理解 PackageReference，现在可以大幅简化
- 删除 DeployCompilerGeneratorToolsRuntime 项目，不再需要
