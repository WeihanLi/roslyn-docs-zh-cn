# Visual C# 编译器选项

| 标志 | 描述 |
| ---- | ---- |
| **输出文件** |
| `/out:`*file* | 指定输出文件名（默认：带有主类的文件或第一个文件的基本名称）
| `/refout:`*file* | 指定引用程序集的输出文件名
| `/target:exe` |  构建控制台可执行文件（默认）（简写形式：`/t:exe`）
| `/target:winexe` | 构建 Windows 可执行文件（简写形式：`/t:winexe`）
| `/target:library` | 构建库（简写形式：`/t:library`）
| `/target:module` | 构建可添加到另一个程序集的模块（简写形式：`/t:module`）
| `/target:appcontainerexe` | 构建 Appcontainer 可执行文件（简写形式：`/t:appcontainerexe`）
| `/target:winmdobj` | 构建由 WinMDExp 使用的 Windows 运行时中间文件（简写形式：`/t:winmdobj`）
| `/doc:`*file* | 要生成的 XML 文档文件
| `/platform:`*string* | 限制此代码可以运行的平台：`x86`、`Itanium`、`x64`、`arm`、`anycpu32bitpreferred` 或 `anycpu`。默认是 `anycpu`。
| **输入文件**
| `/recurse:`*wildcard* | 根据通配符规范包含当前目录和子目录中的所有文件
| `/reference:`*alias*=*file* | 使用给定别名从指定程序集文件引用元数据（简写形式：`/r`）
| `/reference:`*file list* | 从指定的程序集文件引用元数据（简写形式：`/r`）
| `/addmodule:`*file list* | 将指定的模块链接到此程序集
| `/link:`*file list* | 从指定的互操作程序集文件嵌入元数据（简写形式：`/l`）
| `/analyzer:`*file list* | 从此程序集运行分析器（简写形式：`/a`）
| `/additionalfile:`*file list* | 不直接影响代码生成但可能被分析器用于产生错误或警告的附加文件。
| **资源**
| `/win32res:`*file* | 指定 Win32 资源文件（.res）
| `/win32icon:`*file* | 使用此图标作为输出
| `/win32manifest`:*file* | 指定 Win32 清单文件（.xml）
| `/nowin32manifest` | 不包含默认的 Win32 清单
| `/resource`:*resinfo* | 嵌入指定的资源（简写形式：`/res`）
| `/linkresource`:*resinfo* | 将指定的资源链接到此程序集（简写形式：`/linkres`），其中 *resinfo* 格式为 *file*{`,`*string name*{`,``public``|``private`}}
| **代码生成**
| `/debug`{`+`&#124;`-`} | 发出（或不发出）调试信息
| `/debug`:`full` | 使用当前平台的默认格式将调试信息发出到 .pdb 文件：Windows 上为 _Windows PDB_，其他系统上为 _Portable PDB_
| `/debug`:`pdbonly` | 与 `/debug:full` 相同。为了向后兼容。
| `/debug`:`portable` | 使用跨平台 [Portable PDB 格式](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/portable_pdb.md)将调试信息发出到 .pdb 文件
| `/debug`:`embedded` | 使用 [Portable PDB 格式](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/portable_pdb.md)将调试信息嵌入到 .dll/.exe 本身（不生成 .pdb 文件）。
| `/sourcelink`:*file* | 要嵌入到 PDB 中的[源链接](https://github.com/dotnet/core/blob/main/Documentation/diagnostics/source_link.md)信息。
| `/optimize`{`+`&#124;`-`} | 启用优化（简写形式：`/o`）
| `/deterministic` | 生成确定性程序集（包括模块版本 GUID 和时间戳）
| `/refonly` | 生成引用程序集而不是完整程序集作为主要输出
| **错误和警告**
| `/warnaserror`{`+`&#124;`-`} | 将所有警告报告为错误
| `/warnaserror`{`+`&#124;`-`}`:`*warn list* | 将特定警告报告为错误
| `/warn`:*n* | 设置警告级别（非负整数）（简写形式：`/w`）
| `/nowarn`:*warn list* | 禁用特定警告消息
| `/ruleset`:*file* | 指定禁用特定诊断的规则集文件。
| `/errorlog`:*file* | 指定用于记录所有编译器和分析器诊断的文件。
| `/reportanalyzer` | 报告额外的分析器信息，如执行时间。
| **语言**
| `/checked`{`+`&#124;`-`} | 生成溢出检查
| `/unsafe`{`+`&#124;`-`} | 允许 'unsafe' 代码
| `/define:`*symbol list* | 定义条件编译符号（简写形式：`/d`）
| `/langversion:?` | 显示语言版本的允许值
| `/langversion`:*string* | 指定语言版本，如 `default`（最新主要版本）或 `latest`（最新版本，包括次要版本）
| **安全性**
| `/delaysign`{`+`&#124;`-`} | 仅使用强名称密钥的公共部分延迟签名程序集
| `/keyfile:`*file* | 指定强名称密钥文件
| `/keycontainer`:*string* | 指定强名称密钥容器
| `/highentropyva`{`+`&#124;`-`} | 启用高熵 ASLR
| **杂项**
| `@`*file* | 读取响应文件以获取更多选项
| `/help` | 显示使用消息（简写形式：`/?`）
| `/nologo` | 抑制编译器版权消息
| `/noconfig` | 不自动包含 `CSC.RSP` 文件
| `/parallel`{`+`&#124;`-`} | 并发构建。
| **高级**
| `/baseaddress:`*address* | 要构建的库的基地址
| `/bugreport:`*file* | 创建"错误报告"文件
| `/checksumalgorithm:`*alg* | 指定用于计算存储在 PDB 中的源文件校验和的算法。支持的值：`SHA1`（默认）或 `SHA256`。
| `/codepage:`*n* | 指定打开源文件时使用的代码页
| `/utf8output` | 以 UTF-8 编码输出编译器消息
| `/main`:*type* | 指定包含入口点的类型（忽略所有其他可能的入口点）（简写形式：`/m`）
| `/fullpaths` | 编译器生成完全限定路径
| `/filealign`:*n* | 指定用于输出文件节的对齐方式
| `/pathmap:`*k1*=*v1*,*k2*=*v2*,... | 指定编译器输出的源路径名的映射。两个连续的分隔符字符被视为键或值的一部分的单个字符（即 `==` 代表 `=`，`,,` 代表 `,`）。
| `/pdb:`*file* | 指定调试信息文件名（默认：带有 `.pdb` 扩展名的输出文件名）
| `/errorendlocation` | 输出每个错误的结束位置的行和列
| `/preferreduilang` | 指定首选输出语言名称。
| `/nostdlib`{`+`&#124;`-`} | 不引用标准库 `mscorlib.dll`
| `/subsystemversion:`*string* | 指定此程序集的子系统版本
| `/lib:`*file list* | 指定搜索引用的其他目录
| `/errorreport:`*string* | 指定如何处理内部编译器错误：`prompt`、`send`、`queue` 或 `none`。默认是 `queue`。
| `/appconfig:`*file* | 指定包含程序集绑定设置的应用程序配置文件
| `/moduleassemblyname:`*string* | 此模块将成为其一部分的程序集名称
| `/modulename:`*string* | 指定源模块的名称
