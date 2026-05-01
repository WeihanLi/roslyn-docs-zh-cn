每次发布时，需检查以下步骤：

- 开始：在公共分支可用时声明新的语言版本（此步骤有其自己的检查清单）
- 开始：审查/分类该里程碑期间应考虑的 issues（清理未标记的 issues）
- 预览：审查公共 API 变更
- 预览：移除所有未使用的语言版本
- 预览/RTM：通知合作伙伴预览包和 RTM 包已发布到 NuGet
- 预览/RTM：通知文档团队第一个预览版的发布时间和 RTM 时间线（以便发布文档）
- 预览/RTM：更新 Visual Studio 发行说明
- RTM：在团队博客和 Visual Studio Preview 发行说明中宣布新功能
- RTM：记录[语言历史](https://github.com/dotnet/csharplang/blob/main/Language-Version-History.md)，并将[语言提案](https://github.com/dotnet/csharplang/tree/main/proposals)移至适当的文件夹
- 更新包描述

对于每个语言特性：

- 与 LDM 合作确定负责人并定义该特性
- 在开始开发特性时更新[语言特性状态](https://github.com/dotnet/roslyn/blob/main/docs/Language%20Feature%20Status.md)页面
- 确定指定的审查者
- 在有公共 API 变更的 PR 上通知 Jared 和 Neal
- 破坏性变更需要经过兼容性委员会批准并[记录在案](https://github.com/dotnet/roslyn/tree/main/docs/compilers/CSharp)
- 指定的审查者应记录并验证测试计划
- 在将特性合并回 `main` 之前，应识别并解决阻塞性问题（包括解决 `PROTOTYPE` 注释）
- 特性完成后更新特性状态

要添加语言版本：
- 将其添加到编译器中（测试有检查清单）
- 将其添加到项目系统，使其出现在 UI 下拉列表中
- 对于 VB，在内部仓库中为旧版项目系统添加
- 将其添加到 [release.json](https://github.com/dotnet/core/pull/1454)（需要确认）
