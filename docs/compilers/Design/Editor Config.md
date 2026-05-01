.editorconfig 编译器支持
==============================

链接文件或"外部"文件
==========================

编译器没有关于源文件是否被链接或来自外部来源（例如 NuGet 包）的概念。这一设计在 .editorconfig 中同样保留：将 .editorconfig 文件应用于外部文件的唯一机制，是将这些文件映射或复制到文件系统上的适当位置，然后将该位置包含在 .editorconfig 的路径规范中。对于链接文件，这通常可以通过相对路径来实现。例如，假设代码布局如下：

```
Repo Root
  | --- LinkedFile.cs
  | --- Subproject
          | --- .editorconfig
          | --- SourceFile1.cs
          | --- SourceFile2.cs
```

在此示例中，如果子项目的 `.editorconfig` 仅包含 `[*.cs]` 规范，则它不会应用于 `LinkedFile.cs`。如果用户希望明确将链接文件映射进来，使子项目的 .editorconfig 也能应用，则需要添加一个新的 `.editorconfig` 文件，其中包含使用相对路径规范指向该链接文件的节。例如：

```
Repo Root
  | --- .editorconfig
  | --- LinkedFile.cs
  | --- Subproject
          | --- .editorconfig
          | --- SourceFile1.cs
          | --- SourceFile2.cs
```

根目录下的 .editorconfig 需包含：

```
[LinkedFile.cs]
option = value
```

对于外部文件（例如源 NuGet 包提供的文件），可以将 NuGet 包还原目录本身映射到仓库路径中，然后像处理链接文件一样，将相对路径添加到 `.editorconfig` 中。
