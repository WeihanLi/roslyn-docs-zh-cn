# 创建功能分支
本文档描述了在 roslyn 功能分支上设置 CI 的过程。

## 推送分支
第一步是创建一个以 roslyn 上的初始更改为种子的分支。此分支应命名为 `features/<功能名称>`。例如：`features/mono` 用于处理 mono 工作。

假设分支应以 `main` 的内容开始，可以通过执行以下操作创建分支：

注意：这些步骤假设远程 `origin` 指向官方 [roslyn 仓库](https://github.com/dotnet/roslyn)。

``` cmd
> git fetch origin
> git checkout -B init origin/main
> git push origin init:features/mono
```

## 将分支添加到 Azure Pipelines
需要编辑以下文件，以便 GitHub 在 PR 上触发 Azure Pipelines 测试运行：

- [azure-pipelines.yml](https://github.com/dotnet/roslyn/blob/main/azure-pipelines.yml)
- [azure-pipelines-integration.yml](https://github.com/dotnet/roslyn/blob/main/azure-pipelines-integration.yml)

在文件的 `pr` 部分下添加您的分支名称。

``` yaml
pr:
- main
- main-vs-deps
- ...
- features/mono
```
