简介
============

本文档涵盖以下内容：

* 附加文件的用途
* 在命令行上传递附加文件
* 在 MSBuild 项目中将单个项指定为 `AdditionalFile`
* 在 MSBuild 项目中将整组项指定为 `AdditionalFile`
* 通过 `AnalyzerOptions` 类型访问和读取附加文件
* 包含读取附加文件的示例分析器

用途
====

有时分析器需要访问通过正常编译器输入（源文件、引用和选项）不可用的信息。为支持这些场景，C# 和 Visual Basic 编译器可以接受额外的非源文本文件作为输入。

例如，分析器可以强制执行项目中不使用一组禁止术语，或者每个源文件都有某个特定的版权标头。术语或版权标头可以作为附加文件传递给分析器，而不是在分析器本身中硬编码。

在命令行上
===================

在命令行上，可以使用 `/additionalfile` 选项传递附加文件。例如：
```
csc.exe alpha.cs /additionalfile:terms.txt
```

在项目文件中
=================

传递单个文件
--------------------------

要将单个项目项指定为附加文件，请将项类型设置为 `AdditionalFiles`：

``` XML
<ItemGroup>
  <AdditionalFiles Include="terms.txt" />
</ItemGroup>
```

传递一组文件
------------------------

有时无法更改项类型，或者需要将整组项作为附加文件传递。在这种情况下，您可以更新 `AdditionalFileItemNames` 属性以指定要包含的项类型。例如，如果您的分析器需要访问项目中的所有 .resx 文件，您可以执行以下操作：
``` XML
<PropertyGroup>
  <!-- 更新属性以包含所有 EmbeddedResource 文件 -->
  <AdditionalFileItemNames>$(AdditionalFileItemNames);EmbeddedResource</AdditionalFileItemNames>
</PropertyGroup>
<ItemGroup>
  <!-- 现有资源文件 -->
  <EmbeddedResource Include="Terms.resx">
    ...
  </EmbeddedResource>
</ItemGroup>
```

访问附加文件
==========================

附加文件集可以通过传递给诊断操作的上下文对象的 `Options` 属性访问。例如：
```C#
CompilationAnalysisContext context = ...;
ImmutableArray<AdditionalText> additionFiles = context.Options.AdditionalFiles;
```

从 `AdditionalText` 实例，您可以访问文件路径或内容作为 `SourceText`：
``` C#
AdditionText additionalFile = ...;
string path = additionalFile.Path;
SourceText contents = additionalFile.GetText();
```

示例
=======

逐行读取文件
---------------------------

此示例读取一个简单的文本文件，获取一组术语（每行一个），这些术语不应在类型名称中使用。

``` C#
using System.Collections.Generic;
using System.Collections.Immutable;
using System.IO;
using System.Linq;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Diagnostics;
using Microsoft.CodeAnalysis.Text;

[DiagnosticAnalyzer(LanguageNames.CSharp, LanguageNames.VisualBasic)]
public class CheckTermsAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "CheckTerms001";

    private const string Title = "类型名称包含无效术语";
    private const string MessageFormat = "术语 '{0}' 不允许在类型名称中使用。";
    private const string Category = "Policy";

    private static DiagnosticDescriptor Rule =
        new DiagnosticDescriptor(
            DiagnosticId,
            Title,
            MessageFormat,
            Category,
            DiagnosticSeverity.Error,
            isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics { get { return ImmutableArray.Create(Rule); } }

    public override void Initialize(AnalysisContext context)
    {
        context.RegisterCompilationStartAction(compilationStartContext =>
        {
            // 查找包含无效术语的文件。
            ImmutableArray<AdditionalText> additionalFiles = compilationStartContext.Options.AdditionalFiles;
            AdditionalText termsFile = additionalFiles.FirstOrDefault(file => Path.GetFileName(file.Path).Equals("Terms.txt"));

            if (termsFile != null)
            {
                HashSet<string> terms = new HashSet<string>();

                // 逐行读取文件以获取术语。
                SourceText fileText = termsFile.GetText(compilationStartContext.CancellationToken);
                foreach (TextLine line in fileText.Lines)
                {
                    terms.Add(line.ToString());
                }

                // 检查每个命名类型是否包含无效术语。
                compilationStartContext.RegisterSymbolAction(symbolAnalysisContext =>
                {
                    INamedTypeSymbol namedTypeSymbol = (INamedTypeSymbol)symbolAnalysisContext.Symbol;
                    string symbolName = namedTypeSymbol.Name;

                    foreach (string term in terms)
                    {
                        if (symbolName.Contains(term))
                        {
                            symbolAnalysisContext.ReportDiagnostic(Diagnostic.Create(Rule, namedTypeSymbol.Locations[0], term));
                        }
                    }
                },
                SymbolKind.NamedType);
            }
        });
    }
}

```

将文件转换为流
-----------------------------

在附加文件包含结构化数据（例如 XML 或 JSON）的情况下，`SourceText` 提供的逐行访问可能不是所需的。一种替代方法是通过调用 `ToString()` 将 `SourceText` 转换为 `string`。此示例演示了另一种替代方法：将 `SourceText` 转换为 `Stream` 以供其他库使用。假设术语文件具有以下格式：

``` XML
<Terms>
  <Term>frob</Term>
  <Term>wizbang</Term>
  <Term>orange</Term>
</Terms>
```

``` C#
using System.Collections.Generic;
using System.Collections.Immutable;
using System.IO;
using System.Linq;
using System.Xml.Linq;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Diagnostics;
using Microsoft.CodeAnalysis.Text;

[DiagnosticAnalyzer(LanguageNames.CSharp, LanguageNames.VisualBasic)]
public class CheckTermsXMLAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "CheckTerms001";

    private const string Title = "类型名称包含无效术语";
    private const string MessageFormat = "术语 '{0}' 不允许在类型名称中使用。";
    private const string Category = "Policy";

    private static DiagnosticDescriptor Rule =
        new DiagnosticDescriptor(
            DiagnosticId,
            Title,
            MessageFormat,
            Category,
            DiagnosticSeverity.Error,
            isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics { get { return ImmutableArray.Create(Rule); } }

    public override void Initialize(AnalysisContext context)
    {
        context.RegisterCompilationStartAction(compilationStartContext =>
        {
            // 查找包含无效术语的文件。
            ImmutableArray<AdditionalText> additionalFiles = compilationStartContext.Options.AdditionalFiles;
            AdditionalText termsFile = additionalFiles.FirstOrDefault(file => Path.GetFileName(file.Path).Equals("Terms.xml"));

            if (termsFile != null)
            {
                HashSet<string> terms = new HashSet<string>();
                SourceText fileText = termsFile.GetText(compilationStartContext.CancellationToken);

                MemoryStream stream = new MemoryStream();
                using (StreamWriter writer = new StreamWriter(stream, Encoding.UTF8, 1024, true))
                {
                    fileText.Write(writer);
                }
                
                stream.Position = 0;

                // 读取所有 <Term> 元素以获取术语。
                XDocument document = XDocument.Load(stream);
                foreach (XElement termElement in document.Descendants("Term"))
                {
                    terms.Add(termElement.Value);
                }

                // 检查每个命名类型是否包含无效术语。
                compilationStartContext.RegisterSymbolAction(symbolAnalysisContext =>
                {
                    INamedTypeSymbol namedTypeSymbol = (INamedTypeSymbol)symbolAnalysisContext.Symbol;
                    string symbolName = namedTypeSymbol.Name;

                    foreach (string term in terms)
                    {
                        if (symbolName.Contains(term))
                        {
                            symbolAnalysisContext.ReportDiagnostic(Diagnostic.Create(Rule, namedTypeSymbol.Locations[0], term));
                        }
                    }
                },
                SymbolKind.NamedType);
            }
        });
    }
}
```
