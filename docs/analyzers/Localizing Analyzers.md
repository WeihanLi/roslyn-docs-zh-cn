简介
============

本文档涵盖以下内容：
* 设置项目以使用本地化资源
 * 在分析器中使用本地化资源
 * 在代码修复中使用本地化资源
* 生成本地化资源程序集
* 在 NuGet 包和 VSIX 中打包资源程序集
* 在命令行和 VS 中测试本地化

设置项目
=======================

您需要创建一个 .resx 文件来保存您的可本地化资源。在本文档中，我们假设您将它们放在名为 'AnalyzerResources.resx' 的文件中。

在分析器中使用本地化资源
--------------------------------------

通常，.NET 应用程序可以简单地通过从 .resx 文件生成的类型访问资源：
``` C#
string message = AnalyzerResources.Message;
```
框架将使用 `System.Globalization.CultureInfo.CurrentUICulture` 中指定的区域性自动查找并返回适当本地化的资源。

但是，C# 和 VB 编译器提供 `/preferreduilang` 开关来指定使用与 `CurrentUICulture` 不同区域性的资源。因此，在分析器中直接访问 `AnalyzerResources.Message` 将不会返回正确的资源。

相反，您需要通过 `LocalizableResourceString` 类型访问这些资源。例如：
``` C#
LocalizableResourceString message = new LocalizableResourceString(nameof(AnalyzerResources.Message), AnalyzerResources.ResourceManager, typeof(AnalyzerResources));
```

在代码修复中使用本地化资源
---------------------------------------

另一方面，代码修复不受 `/preferreduilang` 开关的影响。代码修复中的资源应以标准方式访问：
``` C#
string codeFixMessage = AnalyzerResources.CodeFixMessage;
```

生成本地化资源程序集
=======================================

要创建资源程序集，您只需添加包含所需[语言名称](https://msdn.microsoft.com/en-us/library/windows/desktop/dd318696(v=vs.85).aspx)的额外 .resx 文件。例如，如果您想创建包含德语字符串的资源程序集，您可以添加 AnalyzerResources.de.resx。对于日语，您添加 AnalyzerResources.ja.resx，依此类推。

在这些文件中，您添加名称与 AnalyzerResources.resx 中的名称匹配的项，值包含本地化内容。如果 AnalyzerResources.resx 包含以下内容：

名称    | 值
--------|-------
Message | Hello!

那么 AnalyzerResources.de.resx 应包含：

名称    | 值
--------|-----------
Message | Guten Tag!

现在当您构建时，资源程序集会自动在语言特定的子文件夹中生成：

- bin\Debug\
  - MyAnalyzer.dll
  - MyAnalyzer.pdb
  - de\
    - MyAnalyzer.resources.dll
  - ja\
    - MyAnalyzer.resources.dll

这种布局也是 CLR 在运行时期望的：对于给定的程序集 Foo.dll，本地化资源应该在程序集旁边的语言特定文件夹中。

打包资源程序集
=============================

NuGet
-----
NuGet 打包很简单，因为您只需复制构建输出的结构：
``` XML
<files>
  <file src="MyAnalyzer.dll" target="tools\analyzers" />
  <file src="de\MyAnalyzer.resources.dll" target="tools\analyzers\de" />
  <file src="ja\MyAnalyzer.resources.dll" target="tools\analyzers\ja" />
  ...
</files>
```

VSIX
----

扩展项目会自动将附属程序集合并到生成的 .vsix 文件中。

测试分析器本地化
=============================

在命令行
-------------------
安装 NuGet 包后，您可以通过向 csc.exe 或 vbc.exe 传递 `/preferreduilang` 标志来强制构建使用本地化资源：

    csc.exe /preferreduilang:de /t:library /analyzer:... Foo.cs
    
或者，您可以将其作为属性传递给 MSBuild：

    msbuild /p:PreferredUILang=de
    
或在 *proj 文件本身中将其设置为属性：

    <PreferredUILang>de</PreferredUILang>

在 VS 中
-----

在 VS 中，分析器使用所选的 UI 语言。可以通过从"工具"菜单中选择"选项..."，然后导航到环境 -> 国际设置来更改。
