简介
============

本文档与[规则集架构](..//..//src//Compilers//Core//Portable//RuleSet//RuleSetSchema.xsd)一起描述了 C# 和 Visual Basic 编译器用于打开和关闭诊断分析器以及控制其严重性的 .ruleset 文件的结构。

本文档仅讨论 .ruleset 文件的必需和常见部分；有关完整集，请参阅架构文件。

示例
=====

以下演示了一个小但完整的 .ruleset 文件示例。

``` XML
<RuleSet Name="Project WizBang Rules"
         ToolsVersion="12.0">
  <Include Path="..\OtherRules.ruleset" 
           Action="Default" />
  <Rules AnalyzerId="System.Runtime.Analyzers"
         RuleNamespace="System.Runtime.Analyzers">
    <Rule Id="CA1027" Action="Warning" />
    <Rule Id="CA1309" Action="Error" />
    <Rule Id="CA2217" Action="Warning" />
  </Rules>
  <Rules AnalyzerId="System.Runtime.InteropServices.Analyzers"
         RuleNamespace="System.Runtime.InteropService.Analyzers">
    <Rule Id="CA1401" Action="None" />
    <Rule Id="CA2101" Action="Error" />
  </Rules>
</RuleSet>
```

向编译器传递规则集
=================================

命令行
------------

可以使用 `/ruleset` 开关将规则集文件传递给 csc.exe 和 vbc.exe，该开关接受绝对或相对文件路径。例如：
```
vbc.exe /ruleset:ProjectRules.ruleset ...
vbc.exe /ruleset:..\..\..\SolutionRules.ruleset ...
vbc.exe /ruleset:D:\WizbangCorp\Development\CompanyRules.ruleset ...
```

MSBuild 项目
----------------

在 MSBuild 项目文件中，可以通过 `CodeAnalysisRuleSet` 属性指定规则集。例如：
``` XML
<PropertyGroup>
  <CodeAnalysisRuleSet>ProjectRules.ruleset</CodeAnalysisRuleSet>
</PropertyGroup>
```

请注意，由于规则集是通过*属性*而不是*项*指定的，因此默认情况下，Visual Studio 等 IDE 不会将规则集作为项目中的文件显示。因此，通常还会将文件显式包含为项：
``` XML
<ItemGroup>
  <None Include="ProjectRules.ruleset" />
</ItemGroup>
```

元素
========

`RuleSet`
---------

`RuleSet` 是文件的根元素。

属性：
* Name（**必需**）：文件的简短描述性名称。
* Description（**可选**）：对规则集文件用途的较长描述。
* ToolsVersion（**必需**）：此属性是与使用 .ruleset 文件的其他工具的向后兼容性所必需的。实际上，它是生成此 .ruleset 文件的 Visual Studio 版本。如有疑问，只需使用 "12.0"。

子元素：`Include`、`Rules`

`Include`
---------

从指定的规则集文件中提取设置。当前文件中的设置会覆盖包含的文件中的设置。

父元素：`RuleSet`

属性：
* Path（**必需**）：另一个 .ruleset 文件的绝对或相对路径。
* Action（**必需**）：指定包含规则的有效操作。必须是以下值之一：
 * Default - 规则使用包含的文件中指定的操作。
 * Error - 包含的规则被视为其操作值都是 "Error"。
 * Warning - 包含的规则被视为其操作值都是 "Warning"。
 * Info - 包含的规则被视为其操作值都是 "Info"。
 * Hidden - 包含的规则被视为其操作值都是 "Hidden"。
 * None - 包含的规则被视为其操作值都是 "None"。

子元素：无。

`Rules`
-------

保存来自单个诊断分析器的规则设置。

父元素：`RuleSet`

属性：
* AnalyzerId（**必需**）：分析器的名称。实际上，这是包含诊断的程序集的简单名称。
* RuleNamespace（**必需**）：此属性是与使用 .ruleset 文件的其他工具的向后兼容性所必需的。实际上，它通常与 AnalyzerId 相同。

子元素：`Rule`

`Rule`
------

指定对给定诊断规则执行的操作。

父元素：`Rules`

属性：
* Id（**必需**）：诊断规则的 ID。
* Action（**必需**）：以下值之一：
 * Error - 此诊断的实例被视为编译器错误。
 * Warning - 此诊断的实例被视为编译器警告。
 * Info - 此诊断的实例被视为编译器消息。
 * Hidden - 此诊断的实例对用户隐藏。
 * None - 关闭诊断规则。

子元素：无。

注意：

在 `Rules` 元素中，具有给定 `Id` 的 `Rule` 只能出现一次。C# 和 Visual Basic 编译器施加了进一步的约束：如果多个 `Rules` 元素包含具有相同 `Id` 的 `Rule`，它们必须都指定相同的 `Action` 值。

Hidden 和 None 操作之间的区别是微妙但重要的。当诊断规则设置为 Hidden 时，该规则的实例仍然会被创建，但默认情况下不会向用户显示。但是，主机进程可以访问这些诊断并执行一些主机特定的操作。例如，Visual Studio 使用 Hidden 诊断和自定义编辑器扩展的组合来在编辑器中"灰显"死代码或不必要的代码。但是，设置为 None 的诊断规则实际上是被关闭的。此诊断的实例可能根本不会产生；即使它们被抑制而不是提供给主机。
