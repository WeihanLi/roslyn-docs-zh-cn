# 跨平台说明

## 注意事项

对于开发 Roslyn 的 Linux 和 Mac 支持仍在进行中。目前并非所有内容都受支持，此页面上详述的步骤将会频繁更改。如果您对此领域感兴趣，请经常检查更新。

## 构建

使用以下命令构建所有跨平台项目：

```
cd <roslyn-git-directory>
./build.sh --restore
```

如果在 `$PATH` 上找不到 .NET Core，该脚本将安装 .NET Core 到 `.dotnet` 目录。然后它将恢复所需的 NuGet 包并构建 `Compilers.slnf` 解决方案。选项 `--restore`（或 `-r`）仅在首次构建时或当 NuGet 引用更改时需要。

## 使用编译器

构建后，`artifacts/bin/csc/Debug/netcoreapp3.1` 和 `artifacts/bin/csc/Debug/net472` 目录中将有一个 `csc`。使用前者在 .NET Core 上运行，使用后者在 Mono 上运行。

### 在 Mono 上运行 `csc.exe`

使用 `mono artifacts/bin/csc/Debug/net472/csc.exe -noconfig` 运行 C# 编译器。

需要 `-noconfig` 是因为 `csc.exe` 默认引用其旁边的 `csc.rsp` 文件。这是 Windows 响应文件，因此在 Mono 上运行时并非所有程序集都存在。传递 `-noconfig` 选项以忽略此响应文件。
