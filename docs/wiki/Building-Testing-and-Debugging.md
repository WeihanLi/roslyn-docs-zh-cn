太好了！我们非常高兴你能加入我们的旅程！在开始之前，**请务必确认你正在正确的分支上工作**。一旦确定了要工作的分支，请按照下表中各分支描述中的相应链接，获取构建/测试/调试的具体说明。

## 选择你的分支
以下是你需要了解的主要分支：

| 分支 |       |
| ------ | ----- | 
| [**main**](https://github.com/dotnet/roslyn/tree/main) | 我们的主要分支。此处的更改当前面向 Visual Studio 2022 17.0。在没有其他指导的情况下，你应该在此分支工作并提交拉取请求。<br/>[Windows 上的构建说明](https://github.com/dotnet/roslyn/blob/main/docs/contributing/Building,%20Debugging,%20and%20Testing%20on%20Windows.md) <br/>[Linux 和 Mac 上的构建说明](https://github.com/dotnet/roslyn/blob/main/docs/infrastructure/cross-platform.md) |
| [**community**](https://github.com/dotnet/roslyn/tree/community) | 如果你看到某个**置顶 issue** 或类似公告引用了 **community** 分支，则你可能需要以该分支而非 main 为基础开展工作。如果点击此分支链接返回 HTTP 404，则可以认为此时无需使用该分支。当 Roslyn 团队需要依赖未发布的组件时，有时会使用此分支，这会导致我们的 main 分支无法与公开发布的 Visual Studio 版本一起使用。  |
## 已知问题
请查看[已知贡献者问题](https://github.com/dotnet/roslyn/labels/Contributor%20Pain)，你在为 Roslyn 做贡献时可能会遇到这些问题。如果你的问题未在列表中，请提交反馈。
