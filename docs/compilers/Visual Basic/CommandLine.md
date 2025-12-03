# Visual Basic 编译器选项

| 标志 | 描述 |
| ---- | ---- |
| **输出文件**
| `/out:`*file* | 指定输出文件名。
| `/refout:`*file* | 指定引用程序集的输出文件名
| `/target:exe` | 创建控制台应用程序（默认）。（短形式：`/t`）
| `/target:winexe` | 创建 Windows 应用程序。
| `/target:library` | 创建库程序集。
| `/target:module` | 创建可添加到程序集的模块。
| `/target:appcontainerexe` | 创建在 AppContainer 中运行的 Windows 应用程序。
| `/target:winmdobj` | 创建 Windows 元数据中间文件
| `/doc`{`+`&#124;`-`} | 生成 XML 文档文件。
| `/doc:`*file* | 生成 XML 文档文件到 *file*。
| **输入文件**
| `/addmodule:`*file_list* | 从指定模块引用元数据
| `/link:`*file_list* | 从指定的互操作程序集嵌入元数据。（短形式：`/l`）
| `/recurse:`*wildcard* | 根据通配符规范包含当前目录和子目录中的所有文件。
| `/reference:`*file_list* | 从指定的程序集引用元数据。（短形式：`/r`）
| `/analyzer:`*file_list* | 从此程序集运行分析器（短形式：`/a`）
| `/additionalfile:`*file list* | 不直接影响代码生成但可由分析器用于生成错误或警告的附加文件。
| **资源**
| `/linkresource`:*resinfo* | 将指定资源链接到此程序集（短形式：`/linkres`）其中 *resinfo* 格式为 *file*{`,`*string name*{`,``public``|``private`}}
| `/resource`:*resinfo* | 嵌入指定资源（短形式：`/res`）
| `/nowin32manifest` | 默认清单不应嵌入到输出 PE 的清单部分。
| `/win32icon:`*file* | 为默认 Win32 资源指定 Win32 图标文件 (.ico)。
| `/win32manifest:`*file* | 提供的文件嵌入到输出 PE 的清单部分。
| `/win32resource:`*file* | 指定 Win32 资源文件 (.res)。
| **代码生成**
| `/debug`{`+`&#124;`-`} | 发出调试信息。
| `/debug`:`full` | 使用当前平台的默认格式将调试信息发出到 .pdb 文件：Windows 上为 _Windows PDB_，其他系统上为 _Portable PDB_
| `/debug`:`pdbonly` | 与 `/debug:full` 相同。为了向后兼容。
| `/debug`:`portable` | 使用跨平台 [Portable PDB 格式](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/portable_pdb.md) 将调试信息发出到 .pdb 文件
| `/debug`:`embedded` | 使用 [Portable PDB 格式](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/portable_pdb.md) 将调试信息嵌入到 .dll/.exe 本身（不生成 .pdb 文件）。
| `/sourcelink`:*file* | 嵌入到 PDB 的 [Source link](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/source_link.md) 信息。
| `/optimize`{`+`&#124;`-`} | 启用优化。
| `/removeintchecks`{`+`&#124;`-`} | 删除整数检查。默认关闭。
| `/debug:full` | 发出完整调试信息（默认）。
| `/debug:portable` | 以便携式格式发出调试信息。
| `/debug:pdbonly` | 仅发出 PDB 文件。
| `/deterministic` | 生成确定性程序集（包括模块版本 GUID 和时间戳）
| `/refonly | 生成引用程序集而不是完整程序集作为主输出
| **错误和警告**
| `/nowarn` | 禁用所有警告。
| `/nowarn:`*number_list* | 禁用单个警告列表。
| `/warnaserror`{`+`&#124;`-`} | 将所有警告视为错误。
| `/warnaserror`{`+`&#124;`-`}:*number_list* | 将警告列表视为错误。
| `/ruleset:`*file* | 指定禁用特定诊断的规则集文件。
| `/errorlog:`*file* | 指定记录所有编译器和分析器诊断的文件。
| `/reportanalyzer` | 报告额外的分析器信息，如执行时间。
| **语言**
| `/define:`*symbol_list* | 声明全局条件编译符号。*symbol_list* 为 *name*`=`*value*`,`...（短形式：`/d`）
| `/imports:`*import_list* | 为引用元数据文件中的命名空间声明全局 Imports。*import_list* 为 *namespace*`,`...
| `/langversion:?` | 显示语言版本的允许值
| `/langversion`:*string* | 指定语言版本，如 `default`（最新主版本）或 `latest`（最新版本，包括次版本）
| `/optionexplicit`{`+`&#124;`-`} | 要求显式声明变量。
| `/optioninfer`{`+`&#124;`-`} | 允许变量的类型推断。
| `/rootnamespace`:*string* | 为所有顶级类型声明指定根命名空间。
| `/optionstrict`{`+`&#124;`-`} | 强制执行严格的语言语义。
| `/optionstrict:custom` | 当不遵守严格的语言语义时警告。
| `/optioncompare:binary` | 指定二进制样式字符串比较。这是默认值。
| `/optioncompare:text` | 指定文本样式字符串比较。
| **杂项**
| `/help` | 显示使用消息。（短形式：`/?`）
| `/noconfig` | 不自动包含 VBC.RSP 文件。
| `/nologo` | 不显示编译器版权横幅。
| `/quiet` | 安静输出模式。
| `/verbose` | 显示详细消息。
| `/parallel`{`+`&#124;`-`} | 并发构建。
| **高级**
| `/baseaddress:`*number* | 库或模块的基地址（十六进制）。
| `/bugreport:`*file* | 创建 bug 报告文件。
| `/checksumalgorithm:`*alg* | 指定用于计算存储在 PDB 中的源文件校验和的算法。支持的值为：`SHA1`（默认）或 `SHA256`。
| `/codepage:`*number* | 指定打开源文件时使用的代码页。
| `/delaysign`{`+`&#124;`-`} | 仅使用强名称密钥的公共部分延迟签名程序集。
| `/errorreport:`*string* | 指定如何处理内部编译器错误；必须为 `prompt`、`send`、`none` 或 `queue`（默认）。
| `/filealign:`*number* | 指定用于输出文件部分的对齐方式。
| `/highentropyva`{`+`&#124;`-`} | 启用高熵 ASLR。
| `/keycontainer:`*string* | 指定强名称密钥容器。
| `/keyfile:`*file* | 指定强名称密钥文件。
| `/libpath:`*path_list* | 搜索元数据引用的目录列表。（分号分隔。）
| `/main:`*class* | 指定包含 Sub Main 的类或模块。也可以是从 System.Windows.Forms.Form 继承的类。（短形式：`/m`）
| `/moduleassemblyname:`*string* | 此模块将成为其一部分的程序集的名称。
| `/netcf` | 面向 .NET Compact Framework。
| `/nostdlib` | 不引用标准库（`system.dll` 和 `VBC.RSP` 文件）。
| `/pathmap:`*k1*=*v1*,*k2*=*v2*,... | 指定编译器输出的源路径名称的映射。两个连续的分隔符字符被视为作为键或值一部分的单个字符（即 `==` 代表 `=`，`,,` 代表 `,`）。
| `/platform:`*string* | 限制此代码可以运行的平台；必须为 `x86`、`x64`、`Itanium`、`arm`、`AnyCPU32BitPreferred` 或 `anycpu`（默认）。
| `/preferreduilang` | 指定首选输出语言名称。
| `/sdkpath:`*path* | .NET Framework SDK 目录的位置（`mscorlib.dll`）。
| `/subsystemversion:`*version* | 指定输出 PE 的子系统版本。*version* 为 *number*{.*number*}
| `/utf8output`{`+`&#124;`-`} | 以 UTF8 字符编码发出编译器输出。
| `@`*file* | 从文本文件插入命令行设置
| `/vbruntime`{+&#124;-&#124;*} | 使用/不使用默认 Visual Basic 运行时编译。
| `/vbruntime:`*file* | 使用 *file* 中的备用 Visual Basic 运行时编译。
