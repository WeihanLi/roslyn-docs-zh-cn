# 增量生成器烹饪书

## 摘要

本文档旨在通过提供常见模式的一系列指南，帮助创建源生成器。
它还旨在阐明在当前设计下可能实现哪些类型的生成器，以及在功能最终设计中明确不在范围内的内容，


**本文档扩展了[完整设计文档](incremental-generators.md)中的细节，请确保先阅读该文档。**

## 目录

- [增量生成器烹饪书](#incremental-generators-cookbook)
  - [摘要](#summary)
  - [目录](#table-of-contents)
  - [提案](#proposal)
  - [不在范围内的设计](#out-of-scope-designs)
    - [语言功能](#language-features)
    - [代码重写](#code-rewriting)
  - [约定](#conventions)
    - [管道模型设计](#pipeline-model-design)
    - [使用 `ForAttributeWithMetadataName`](#use-forattributewithmetadataname)
    - [使用缩进文本写入器而非 `SyntaxNode` 进行生成](#use-an-indented-text-writer-not-syntaxnodes-for-generation)
    - [在生成的标记类型上放置 `Microsoft.CodeAnalysis.EmbeddedAttribute`](#put-microsoftcodeanalysisembeddedattribute-on-generated-marker-types)
    - [不要扫描间接实现接口、间接继承类型或被接口或基类型属性间接标记的类型](#do-not-scan-for-types-that-indirectly-implement-interfaces-indirectly-inherit-from-types-or-are-indirectly-marked-by-an-attribute-from-an-interface-or-base-type)
  - [设计](#designs)
    - [生成类](#generated-class)
    - [附加文件转换](#additional-file-transformation)
    - [增强用户代码](#augment-user-code)
    - [发出诊断](#issue-diagnostics)
    - [INotifyPropertyChanged](#inotifypropertychanged)
    - [将生成器打包为 NuGet 包](#package-a-generator-as-a-nuget-package)
    - [使用 NuGet 包中的功能](#use-functionality-from-nuget-packages)
    - [访问分析器配置属性](#access-analyzer-config-properties)
    - [使用 MSBuild 属性和元数据](#consume-msbuild-properties-and-metadata)
    - [生成器的单元测试](#unit-testing-of-generators)
    - [自动接口实现](#auto-interface-implementation)
  - [重大更改：](#breaking-changes)
  - [待解决问题](#open-issues)

## 提案

作为提醒，源生成器的高级设计目标是：

- 生成器生成一个或多个代表要添加到编译中的 C# 源代码的字符串。
- 明确只支持_添加_操作。生成器可以向编译添加新的源代码，但**不能**修改用户现有的代码。
- 可以访问_附加文件_，即非 C# 源文本。
- _无序_运行，每个生成器将看到相同的输入编译，无法访问其他源生成器创建的文件。
- 用户通过程序集列表指定要运行的生成器，类似于分析器。
- 生成器创建管道，从基本输入源开始，将它们映射到希望生成的输出。暴露的可正确判等的状态越多，编译器就能越早截断更改并重用相同的输出。

## 不在范围内的设计

我们将简要了解不可解决的问题，作为源生成器*不*旨在解决的问题类型的示例：

### 语言功能

源生成器并非设计为替换新的语言功能：例如，可以想象将 [record](records.md) 实现为源生成器，将指定的语法转换为可编译的 C# 表示。

我们明确认为这是一种反模式；语言将继续演进并添加新功能，我们不希望源生成器成为实现这一目的的方式。这样做会创建与不带生成器的编译器不兼容的新 C# “方言”。此外，由于生成器在设计上无法相互交互，以这种方式实现的语言功能很快就会与语言的其他新增功能不兼容。

### 代码重写

用户目前对其程序集执行许多后处理任务，我们在此广泛地将其定义为“代码重写”。这些任务包括但不限于：

- 优化
- 日志注入
- IL 织入
- 调用点重写

虽然这些技术有许多有价值的用例，但它们不符合*源生成*的概念。根据定义，它们是代码修改操作，而源生成器提案明确排除了这些操作。

目前已有成熟的工具和技术来实现这类操作，源生成器提案并不旨在替代它们。我们正在探索调用点重写的方法（参见 [interceptors.md](interceptors.md)），但这些功能是实验性的，可能会有重大变化，甚至被删除。

## 约定

### 管道模型设计

作为一般准则，源生成器管道需要传递_值可判等_的模型。这对于 `IIncrementalGenerator` 的增量性至关重要；一旦管道步骤返回与上次运行相同的信息，生成器驱动程序就可以停止运行生成器，并重用生成器上次生成的相同缓存数据。大多数情况下，当生成器被触发时（尤其是需要使用 `ForAttributeWithMetadataName` 查看类型或方法定义的生成器），触发生成器的编辑实际上不会影响生成器所查看的内容。但是，由于语义几乎可以在任何编辑时改变，生成器驱动程序_必须_重新运行您的生成器以确保这一点。如果您的生成器随后生成与以前相同值的模型，这将使管道短路，避免大量工作。以下是一些关于设计模型以确保维护此等式的一般准则：

* 使用 `record` 而不是 `class`，以便自动生成值相等性。
* 符号（`ISymbol` 及其继承的任何接口）永远不可判等，将它们包含在模型中可能会根植旧的编译，并迫使 Roslyn 保留大量本可释放的内存。永远不要将这些放入模型类型中。应从检查的符号中提取所需信息，转换为可判等的表示：`string` 在这里通常效果很好。
* `SyntaxNode` 在多次运行间通常也不可判等。与符号相比，在管道的初始阶段使用它们并不那么强烈不推荐，[后面](#access-analyzer-config-properties)的示例显示了需要将 `SyntaxNode` 包含在模型中的情况。它们也不像符号那样可能根植大量内存。但是，文件中的任何编辑都将确保该文件中的所有 `SyntaxNode` 不再可判等，因此应尽快将它们从模型中移除。
* 上述要点同样适用于 `Location`。
* 注意模型中的集合类型。.NET 中大多数内置集合类型默认不进行值相等性比较。例如，数组、`ImmutableArray<T>` 和 `List<T>` 使用引用相等性，而不是值相等性。我们建议大多数生成器作者使用数组的包装类型来增强基于值的相等性。

### 使用 `ForAttributeWithMetadataName`

我们强烈建议所有需要检查语法的生成器作者使用标记属性来指示需要检查的类型或成员。这对您作为作者以及您的用户都有多重好处：

* 作为作者，您可以使用 `SyntaxProvider.ForAttributeWithMetadataName`。此实用方法至少比 `SyntaxProvider.CreateSyntaxProvider` 高效 99 倍，在很多情况下甚至更高效。这将帮助您避免在编辑器中给用户造成性能问题。
* 您的用户可以清楚地表明他们_打算_使用您的源生成器。这种意图对于设计良好的用户体验非常有帮助；这意味着您可以编写 Roslyn 分析器，在用户打算使用生成器但以某种方式违反了规则时为其提供帮助。例如，如果您正在生成某个方法体，并且您的生成器要求用户返回特定类型，`GenerateMe` 属性的存在意味着您可以编写一个分析器，在用户的方法声明返回了不应该返回的内容时通知用户。

### 使用缩进文本写入器而非 `SyntaxNode` 进行生成

我们不建议在为 `AddSource` 生成语法时使用 `SyntaxNode`。这样做可能很复杂，且难以良好地格式化；调用 `NormalizeWhitespace` 通常非常昂贵，该 API 并非真正为此用例设计。此外，为确保不可变性保证，`AddSource` 不接受 `SyntaxNode`。它需要获取 `string` 表示形式并将其放入 `SourceText` 中。我们建议使用 `SyntaxNode` 替代方案：使用 `StringBuilder` 的包装器，该包装器跟踪缩进级别，并在调用 `AppendLine` 时在前面添加适当数量的缩进。请参阅[此](https://github.com/dotnet/roslyn/issues/52914#issuecomment-1732680995)关于 `NormalizeWhitespace` 性能的讨论，获取更多示例、性能测量，以及我们为何不认为 `SyntaxNode` 适合此用例的讨论。

### 在生成的标记类型上放置 `Microsoft.CodeAnalysis.EmbeddedAttribute`

用户可能会在同一解决方案的多个项目中依赖您的生成器，而这些项目通常会应用 `InternalsVisibleTo`。这意味着您的 `internal` 标记属性可能在多个项目中定义，编译器会对此发出警告。虽然这不会阻止编译，但会让用户感到烦恼。为避免这种情况，请使用 `Microsoft.CodeAnalysis.EmbeddedAttribute` 标记此类属性；当编译器在来自单独程序集或项目的类型上看到此属性时，将不会将该类型包含在查找结果中。为确保 `Microsoft.CodeAnalysis.EmbeddedAttribute` 在编译中可用，请在 `RegisterPostInitializationOutput` 回调中调用 `AddEmbeddedAttributeDefinition` 辅助方法。

另一个选择是在 NuGet 包中提供定义标记属性的程序集，但这可能更难编写。我们推荐 `EmbeddedAttribute` 方法，除非您需要支持低于 4.14 的 Roslyn 版本。

### 不要扫描间接实现接口、间接继承类型或被接口或基类型属性间接标记的类型

使用接口/基类型标记对生成器来说可能非常诱人且自然。但是，扫描这些类型的标记_非常_昂贵，且无法增量执行。这样做可能对 IDE 和命令行性能产生巨大影响，即使对于相当小的使用项目也是如此。这些场景包括：

* 用户在 `BaseModelType` 上实现接口，然后生成器查找来自 `BaseModelType` 的所有派生类型。由于生成器无法提前知道 `BaseModelType` 实际是什么，这意味着生成器必须对编译中的每种类型获取 `AllInterfaces`，以便扫描标记接口。这最终会在每次按键或每次文件保存时发生，具体取决于用户运行生成器的模式；两者都会对 IDE 性能造成灾难性影响，即使尝试通过将扫描范围缩小到只有基列表的类型来优化也是如此。
* 用户从生成器定义的 `BaseSerializerType` 继承，生成器查找直接或间接继承自该类型的任何内容。与上述场景类似，生成器需要扫描整个编译中具有基类型的所有类型以查找继承的 `BaseSerializerType`，这将严重影响 IDE 性能。
* 生成器在所有基类型/已实现接口中查找具有生成器标记属性的类型。这实际上是场景 1 或 2，只是搜索条件不同。
* 生成器使其标记属性未密封，并期望用户能够从该标记派生自己的属性，作为参数自定义的来源。这有几个问题：首先，每个带属性的类型都需要检查该属性是否继承自标记属性。虽然对性能的影响不如前三个场景大，但对性能并不友好。其次，更重要的是，无法从继承的属性中检索任何自定义。这些属性不是由源生成器实例化的，因此传递给 `base()` 构造函数调用的任何参数或分配给基属性任何属性的值都对生成器不可见。在这里优先使用 FAWMN 驱动的开发，并使用分析器在用户需要从某个基类继承以使生成器正常工作时通知用户。

## 设计

本节按用户场景划分，先列出一般解决方案，然后是更具体的示例。

### 生成类

**用户场景：** 作为生成器作者，我希望能够向编译添加类型，该类型可以被用户代码引用。常见用例包括创建将用于驱动其他源生成器步骤的属性。

**解决方案：** 让用户编写代码，就好像该类型已经存在一样。使用 `RegisterPostInitializationOutput` 步骤，根据编译中可用的信息生成缺失的类型。

**示例：**

给定以下用户代码：

```csharp
public partial class UserClass
{
    [GeneratedNamespace.GeneratedAttribute]
    public partial void UserMethod();
}
```

创建一个生成器，在运行时创建缺失的类型：

```csharp
[Generator]
public class CustomGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static postInitializationContext => {
            postInitializationContext.AddEmbeddedAttributeDefinition();
            postInitializationContext.AddSource("myGeneratedFile.cs", SourceText.From("""
                using System;
                using Microsoft.CodeAnalysis;

                namespace GeneratedNamespace
                {
                    [Embedded]
                    internal sealed class GeneratedAttribute : Attribute
                    {
                    }
                }
                """, Encoding.UTF8));
        });
    }
}
```

**替代解决方案**：如果您除了源生成器外还为用户提供库，只需让该库包含属性定义即可。

### 附加文件转换

**用户场景：** 作为生成器作者，我希望能够将外部非 C# 文件转换为等效的 C# 表示形式。

**解决方案：** 使用 `AdditionalTextsProvider` 过滤并检索文件。将其转换为您关心的代码，然后将该代码注册到解决方案中。

**示例：**

```csharp
[Generator]
public class FileTransformGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var pipeline = context.AdditionalTextsProvider
            .Where(static (text) => text.Path.EndsWith(".xml"))
            .Select(static (text, cancellationToken) =>
            {
                var name = Path.GetFileName(text.Path);
                var code = MyXmlToCSharpCompiler.Compile(text.GetText(cancellationToken));
                return (name, code);
            });

        context.RegisterSourceOutput(pipeline,
            static (context, pair) => 
                // Note: this AddSource is simplified. You will likely want to include the path in the name of the file to avoid
                // issues with duplicate file names in different paths in the same project.
                context.AddSource($"{pair.name}generated.cs", SourceText.From(pair.code, Encoding.UTF8)));
    }
}
```

需要通过使用 `AdditionalFiles` ItemGroup 将项目包含在 csproj 文件中：

```xml
<ItemGroup>
    <AdditionalFiles Include="file1.xml" />
    <AdditionalFiles Include="file2.xml" />
<ItemGroup>
```

### 增强用户代码

**用户场景：** 作为生成器作者，我希望能够检查并用新功能增强用户的代码。

**解决方案：** 要求用户将您想要增强的类设置为 `partial class`，并使用唯一属性标记它。在 `RegisterPostInitializationOutput` 步骤中提供该属性。使用 `ForAttributeWithMetadataName` 注册该属性的回调，以收集生成代码所需的信息，并使用元组（或创建可判等模型）传递该信息。该信息应从语法和符号中提取；**不要将语法或符号放入模型中**。

**示例：**

```csharp
public partial class UserClass
{
    [GeneratedNamespace.Generated]
    public partial void UserMethod();
}
```

```csharp
[Generator]
public class AugmentingGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static postInitializationContext =>
            postInitializationContext.AddEmbeddedAttributeDefinition();
            postInitializationContext.AddSource("myGeneratedFile.cs", SourceText.From("""
                using System;
                using Microsoft.CodeAnalysis;
                namespace GeneratedNamespace
                {
                    [AttributeUsage(AttributeTargets.Method), Embedded]
                    internal sealed class GeneratedAttribute : Attribute
                    {
                    }
                }
                """, Encoding.UTF8)));

        var pipeline = context.SyntaxProvider.ForAttributeWithMetadataName(
            fullyQualifiedMetadataName: "GeneratedNamespace.GeneratedAttribute",
            predicate: static (syntaxNode, cancellationToken) => syntaxNode is BaseMethodDeclarationSyntax,
            transform: static (context, cancellationToken) =>
            {
                var containingClass = context.TargetSymbol.ContainingType;
                return new Model(
                    // Note: this is a simplified example. You will also need to handle the case where the type is in a global namespace, nested, etc.
                    Namespace: containingClass.ContainingNamespace?.ToDisplayString(SymbolDisplayFormat.FullyQualifiedFormat.WithGlobalNamespaceStyle(SymbolDisplayGlobalNamespaceStyle.Omitted)),
                    ClassName: containingClass.Name,
                    MethodName: context.TargetSymbol.Name);
            }
        );

        context.RegisterSourceOutput(pipeline, static (context, model) =>
        {
            var sourceText = SourceText.From($$"""
                namespace {{model.Namespace}};
                partial class {{model.ClassName}}
                {
                    partial void {{model.MethodName}}()
                    {
                        // generated code
                    }
                }
                """, Encoding.UTF8);

            context.AddSource($"{model.ClassName}_{model.MethodName}.g.cs", sourceText);
        });
    }

    private record Model(string Namespace, string ClassName, string MethodName);
}
```

### 发出诊断

**用户场景：** 作为生成器作者，我希望能够向用户的编译添加诊断信息。

**解决方案：** 我们不建议在生成器中发出诊断信息。虽然可以这样做，但在不破坏增量性的情况下进行此操作是超出本烹饪书范围的高级主题。我们建议编写单独的分析器来报告诊断信息。

### INotifyPropertyChanged

**用户场景：** 作为生成器作者，我希望能够自动为用户实现 `INotifyPropertyChanged` 模式。

**解决方案：** 设计原则“明确只支持添加操作”似乎与实现此功能的能力直接矛盾，似乎需要修改用户代码。但是，我们可以利用显式字段，而不是*编辑*用户属性，直接为列出的字段提供它们。

**示例：**

给定如下用户类：

```csharp
using AutoNotify;

public partial class UserClass
{
    [AutoNotify]
    private bool _boolProp;

    [AutoNotify(PropertyName = "Count")]
    private int _intProp;
}
```

生成器可以生成以下内容：

```csharp
using System;
using System.ComponentModel;

namespace AutoNotify
{
    [AttributeUsage(AttributeTargets.Field, Inherited = false, AllowMultiple = false)]
    sealed class AutoNotifyAttribute : Attribute
    {
        public AutoNotifyAttribute()
        {
        }
        public string PropertyName { get; set; }
    }
}


public partial class UserClass : INotifyPropertyChanged
{
    public bool BoolProp
    {
        get => _boolProp;
        set
        {
            _boolProp = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("UserBool"));
        }
    }

    public int Count
    {
        get => _intProp;
        set
        {
            _intProp = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("Count"));
        }
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```

### 将生成器打包为 NuGet 包

**用户场景**：作为生成器作者，我希望将生成器打包为 NuGet 包供使用。

**解决方案：** 生成器可以使用与分析器相同的方法进行打包。确保将生成器放在包的 `analyzers\dotnet\cs` 文件夹中，以便在安装时自动添加到用户的项目中。

例如，要在构建时将生成器项目转换为 NuGet 包，请将以下内容添加到项目文件中：

```xml
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- Generates a package at build -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- Do not include the generator as a lib dependency -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Package the generator in the analyzer directory of the nuget package -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
```

### 使用 NuGet 包中的功能

**用户场景：** 作为生成器作者，我希望在生成器中依赖 NuGet 包提供的功能。

**解决方案：** 可以在生成器中依赖 NuGet 包，但需要对分发进行特殊考虑。

任何*运行时*依赖项（即最终用户程序需要依赖的代码）都可以通过通常的引用机制简单地作为生成器 NuGet 包的依赖项添加。

例如，考虑一个创建依赖 `Newtonsoft.Json` 的代码的生成器。生成器本身不直接使用该依赖项，它只是发出依赖于在用户编译中引用该库的代码。作者将 `Newtonsoft.Json` 作为公共依赖项添加引用，当用户添加生成器包时，它将自动被引用。

```xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- Generates a package at build -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- Do not include the generator as a lib dependency -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Take a public dependency on Json.Net. Consumers of this generator will get a reference to this package -->
    <PackageReference Include="Newtonsoft.Json" Version="12.0.1" />

    <!-- Package the generator in the analyzer directory of the nuget package -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

但是，任何*生成时*依赖项（即生成器在运行和生成代码时使用的依赖项）必须直接与生成器程序集一起打包在生成器 NuGet 包中。没有自动的设施来实现这一点，您需要手动指定要包含的依赖项。

考虑一个在生成过程中使用 `Newtonsoft.Json` 将内容编码为 JSON 的生成器，但不发出任何在运行时依赖于该库存在的代码。作者将添加对 `Newtonsoft.Json` 的引用，但将其所有资产设为*私有*；这确保生成器的使用者不会继承对该库的依赖关系。

作者随后必须将 `Newtonsoft.Json` 库与生成器一起打包在 NuGet 包中。这可以通过以下方式实现：通过添加 `GeneratePathProperty=\"true\"` 设置依赖项以生成路径属性。这将创建格式为 `PKG<PackageName>` 的新 MSBuild 属性，其中 `<PackageName>` 是包名称，其中 `.` 替换为 `_`。在我们的示例中，将有一个名为 `PKGNewtonsoft_Json` 的 MSBuild 属性，其值指向磁盘上 NuGet 文件的二进制内容路径。然后我们可以用它将二进制文件添加到生成的 NuGet 包中，就像我们对生成器本身所做的那样：

```xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- Generates a package at build -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- Do not include the generator as a lib dependency -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Take a private dependency on Newtonsoft.Json (PrivateAssets=all) Consumers of this generator will not reference it.
         Set GeneratePathProperty=true so we can reference the binaries via the PKGNewtonsoft_Json property -->
    <PackageReference Include="Newtonsoft.Json" Version="12.0.1" PrivateAssets="all" GeneratePathProperty="true" />

    <!-- Package the generator in the analyzer directory of the nuget package -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- Package the Newtonsoft.Json dependency alongside the generator assembly -->
    <None Include="$(PkgNewtonsoft_Json)\lib\netstandard2.0\*.dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

```C#
[Generator]
public class JsonUsingGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var pipeline = context.AdditionalTextsProvider.Select(static (text, cancellationToken) =>
        {
            if (!text.Path.EndsWith("*.json"))
            {
                return default;
            }

            return (Name: Path.GetFileName(text.Path), Value: Newtonsoft.Json.JsonConvert.DeserializeObject<MyObject>(text.GetText(cancellationToken).ToString()));
        })
        .Where((pair) => pair is not ((_, null) or (null, _)));

        context.RegisterSourceOutput(pipeline, static (context, pair) =>
        {
            var sourceText = SourceText.From($$"""
                namespace GeneratedNamespace
                {
                    internal sealed class GeneratedClass
                    {
                        public static const (int A, int B) SerializedContent = ({{pair.A}}, {{pair.B}});
                    }
                }
                """, Encoding.UTF8);

            context.AddSource($"{pair.Name}generated.cs", sourceText)
        });
    }

    record MyObject(int A, int B);
}
```

### 访问分析器配置属性

**用户场景：**

- 作为生成器作者，我希望访问语法树或附加文件的分析器配置属性。
- 作为生成器作者，我希望访问自定义生成器输出的键值对。
- 作为生成器用户，我希望能够自定义生成的代码并覆盖默认值。

**解决方案**：生成器可以通过 `AnalyzerConfigOptionsProvider` 访问分析器配置值。分析器配置值可以在 `SyntaxTree`、`AdditionalFile` 的上下文中访问，也可以通过 `GlobalOptions` 全局访问。全局选项是“环境性的”，不适用于任何特定上下文，但在请求特定上下文中的选项时将被包含。

请注意，这是少数几种需要将 `SyntaxNode` 放入管道的情况之一，因为您需要树来获取生成器选项。尽快从管道中移除 `SyntaxNode` 以避免模型无法正确判等。

生成器可以自由使用全局选项自定义其输出。例如，考虑一个可选发出日志记录的生成器。作者可以选择检查全局分析器配置值的值，以控制是否发出日志记录代码。用户随后可以通过 `.globalconfig` 文件按项目选择启用该设置：

```.globalconfig
mygenerator_emit_logging = true
```

```csharp
[Generator]
public class MyGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {

        var userCodePipeline = context.SyntaxProvider.ForAttributeWithMetadataName(... /* collect user code info */);
        var emitLoggingPipeline = context.AnalyzerConfigOptionsProvider.Select(static (options, cancellationToken) =>
            options.GlobalOptions.TryGetValue("mygenerator_emit_logging", out var emitLoggingSwitch)
                ? emitLoggingSwitch.Equals("true", StringComparison.InvariantCultureIgnoreCase)
                : false); // Default

        context.RegisterSourceOutput(userCodePipeline.Combine(emitLoggingPipeline), (context, pair) => /* emit code */);
    }
}
```

### 使用 MSBuild 属性和元数据

**用户场景：**

- 作为生成器作者，我希望根据项目文件中包含的值做出决策
- 作为生成器用户，我希望能够自定义生成的代码并覆盖默认值。

**解决方案：** MSBuild 将自动将指定的属性和元数据转换为可由生成器读取的全局分析器配置。生成器作者通过向 `CompilerVisibleProperty` 和 `CompilerVisibleItemMetadata` 项目组添加项目来指定他们希望提供的属性和元数据。在将生成器打包为 NuGet 包时，可以通过 props 或 targets 文件添加这些内容。

例如，考虑一个基于附加文件创建源代码的生成器，它希望允许用户通过项目文件启用或禁用日志记录。作者将在其 props 文件中指定他们希望使指定的 MSBuild 属性对编译器可见：

```xml
<ItemGroup>
    <CompilerVisibleProperty Include="MyGenerator_EnableLogging" />
</ItemGroup>
```

`MyGenerator_EnableLogging` 属性的值随后将在构建前发送到生成的分析器配置文件，名称为 `build_property.MyGenerator_EnableLogging`。然后生成器可以通过 `AnalyzerConfigOptionsProvider` 管道的 `GlobalOptions` 属性读取此属性：

```c#
context.AnalyzerConfigOptionsProvider.Select((provider, ct) =>
    provider.GlobalOptions.TryGetValue("build_property.MyGenerator_EnableLogging", out var emitLoggingSwitch)
        ? emitLoggingSwitch.Equals("true", StringComparison.InvariantCultureIgnoreCase) : false);
```

因此，用户可以通过在项目文件中设置属性来启用或禁用日志记录。

现在，考虑生成器作者希望可选地允许按附加文件基础选择启用/禁用日志记录。作者可以通过向 `CompilerVisibleItemMetadata` 项目组添加内容，请求 MSBuild 为指定文件发送元数据的值。作者指定他们想要从中读取元数据的 MSBuild itemType（在本例中为 `AdditionalFiles`）以及他们想要为其检索的元数据名称。

```xml
<ItemGroup>
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="MyGenerator_EnableLogging" />
</ItemGroup>
```

`MyGenerator_EnableLogging` 的此值将被发送到生成的分析器配置文件，针对编译中的每个附加文件，项目名称为 `build_metadata.AdditionalFiles.MyGenerator_EnableLogging`。生成器可以在每个附加文件的上下文中读取此值：

```cs
context.AdditionalTextsProvider
       .Combine(context.AnalyzerConfigOptionsProvider)
       .Select((pair, ctx) =>
           pair.Right.GetOptions(pair.Left).TryGetValue("build_metadata.AdditionalFiles.MyGenerator_EnableLogging", out var perFileLoggingSwitch)
               ? perFileLoggingSwitch : false);
```

在用户的项目文件中，用户现在可以为各个附加文件添加注释，说明他们是否想要启用日志记录：

```xml
<ItemGroup>
    <AdditionalFiles Include="file1.txt" />  <!-- logging will be controlled by default, or global value -->
    <AdditionalFiles Include="file2.txt" MyGenerator_EnableLogging="true" />  <!-- always enable logging for this file -->
    <AdditionalFiles Include="file3.txt" MyGenerator_EnableLogging="false" /> <!-- never enable logging for this file -->
</ItemGroup>
```

请注意，通过 `CompilerVisibleProperty` 传递给源生成器的 MSBuild 属性会被写入 editorconfig 文件并从中读取，[对于非简单属性值会导致数据丢失](https://github.com/dotnet/roslyn/issues/51692)。一个可能的解决方法是使用构建任务应用传输编码以防止数据丢失；例如，可以将分号分隔的列表转换为空格分隔的列表（`;` 是 editorconfig 注释字符）：

```xml
<Project>
    <ItemGroup>
        <CompilerVisibleProperty Include="_MyInterpolatorsNamespaces" />
    </ItemGroup>
    <Task Name="_MyInterpolatorsNamespaces" BeforeTargets="BeforeBuild">
        <PropertyGroup>
            <_MyInterpolatorsNamespaces>$([System.String]::Copy('$(InterpolatorsNamespaces)').Replace(';', ' '))</_MyInterpolatorsNamespaces>
        </PropertyGroup>
    </Task>
</Project>
```

**完整示例：**

MyGenerator.props：

```xml
<Project>
    <ItemGroup>
        <CompilerVisibleProperty Include="MyGenerator_EnableLogging" />
        <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="MyGenerator_EnableLogging" />
    </ItemGroup>
</Project>
```

MyGenerator.csproj：

```xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- Generates a package at build -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- Do not include the generator as a lib dependency -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Package the generator in the analyzer directory of the nuget package -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- Package the props file -->
    <None Include="MyGenerator.props" Pack="true" PackagePath="build" Visible="false" />
  </ItemGroup>
</Project>
```

MyGenerator.cs：

```csharp
[Generator]
public class MyGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var emitLoggingPipeline = context.AdditionalTextsProvider
            .Combine(context.AnalyzerConfigOptionsProvider)
            .Select((pair, ctx) =>
                pair.Right.GetOptions(pair.Left).TryGetValue("build_metadata.AdditionalFiles.MyGenerator_EnableLogging", out var perFileLoggingSwitch)
                ? perFileLoggingSwitch.Equals("true", StringComparison.OrdinalIgnoreCase)
                : pair.Right.GlobalOptions.TryGetValue("build_property.MyGenerator_EnableLogging", out var emitLoggingSwitch)
                  ? emitLoggingSwitch.Equals("true", StringComparison.OrdinalIgnoreCase)
                  : false);

        var sourcePipeline = context.AdditionalTextsProvider.Select((file, ctx) => /* Gather build info */);

        context.RegisterSourceOutput(sourcePipeline.Combine(emitLoggingPipeline), (context, pair) => /* Add source */);
    }
}
```

### 生成器的单元测试

**用户场景**：作为生成器作者，我希望能够对生成器进行单元测试，以使开发更容易并确保正确性。

**解决方案 A**：

推荐的方法是使用 [Microsoft.CodeAnalysis.Testing](https://github.com/dotnet/roslyn-sdk/tree/main/src/Microsoft.CodeAnalysis.Testing#microsoftcodeanalysistesting) 包：

- `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing`
- `Microsoft.CodeAnalysis.VisualBasic.SourceGenerators.Testing`

TODO: https://github.com/dotnet/roslyn/issues/72149

### 自动接口实现

**用户场景：** 作为生成器作者，我希望能够自动为用户实现作为类装饰器参数传递的接口的属性

**解决方案：** 要求用户使用 `[AutoImplement]` 属性装饰类，并将他们想要自行实现的接口类型作为参数传递；实现该属性的类必须是 `partial class`。在 `RegisterPostInitializationOutput` 步骤中提供该属性。使用 fullyQualifiedMetadataName `FullyQualifiedAttributeName` 通过 `ForAttributeWithMetadataName` 注册类的回调，并使用元组（或创建可判等模型）传递该信息。该属性也适用于结构体，示例为了工作簿示例的简洁性而故意保持简单。

**Example:**

```csharp
public interface IUserInterface
{
    int InterfaceProperty { get; set; }
}

public interface IUserInterface2
{
    float InterfacePropertyOnlyGetter { get; }
}

[AutoImplementProperties(typeof(IUserInterface), typeof(IUserInterface2))]
public partial class UserClass
{
    public string UserProp { get; set; }
}
```

```csharp
#nullable enable
[Generator]
public class AutoImplementGenerator : IIncrementalGenerator
{
    private const string AttributeNameSpace = "AttributeGenerator";
    private const string AttributeName = "AutoImplementProperties";
    private const string AttributeClassName = $"{AttributeName}Attribute";
    private const string FullyQualifiedAttributeName = $"{AttributeNameSpace}.{AttributeClassName}";

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
        {
            ctx.AddEmbeddedAttributeDefinition();
            //Generate the AutoImplementProperties Attribute
            const string autoImplementAttributeDeclarationCode = $$"""
// <auto-generated/>
using System;
using Microsoft.CodeAnalysis;
namespace {{AttributeNameSpace}};

[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = false), Embedded]
internal sealed class {{AttributeClassName}} : Attribute
{
    public Type[] InterfacesTypes { get; }
    public {{AttributeClassName}}(params Type[] interfacesTypes)
    {
        InterfacesTypes = interfacesTypes;
    }
}
""";
            ctx.AddSource($"{AttributeClassName}.g.cs", autoImplementAttributeDeclarationCode);
        });

        IncrementalValuesProvider<ClassModel> provider = context.SyntaxProvider.ForAttributeWithMetadataName(
            fullyQualifiedMetadataName: FullyQualifiedAttributeName,
            predicate: static (node, cancellationToken_) => node is ClassDeclarationSyntax,
            transform: static (ctx, cancellationToken) =>
            {
                ISymbol classSymbol = ctx.TargetSymbol;

                return new ClassModel(
                    classSymbol.Name,
                    classSymbol.ContainingNamespace.ToDisplayString(),
                    GetInterfaceModels(ctx.Attributes[0])
                    );
            });

        context.RegisterSourceOutput(provider, static (context, classModel) =>
        {
            foreach (InterfaceModel interfaceModel in classModel.Interfaces)
            {
                StringBuilder sourceBuilder = new($$"""
                    // <auto-generated/>
                    namespace {{classModel.NameSpace}};

                    public partial class {{classModel.Name}} : {{interfaceModel.FullyQualifiedName}}
                    {

                    """);

                foreach (string property in interfaceModel.Properties)
                {
                    sourceBuilder.AppendLine(property);
                }

                sourceBuilder.AppendLine("""
                    }
                    """);

                //Concat class name and interface name to have unique file name if a class implements two interfaces with AutoImplement Attribute
                string generatedFileName = $"{classModel.Name}_{interfaceModel.FullyQualifiedName}.g.cs";
                context.AddSource(generatedFileName, sourceBuilder.ToString());
            }
        });
    }

    private static EquatableList<InterfaceModel> GetInterfaceModels(AttributeData attribute)
    {
        EquatableList<InterfaceModel> ret = [];

        if (attribute.ConstructorArguments.Length == 0)
            return ret;

        foreach(TypedConstant constructorArgumentValue in attribute.ConstructorArguments[0].Values)
        {
            if (constructorArgumentValue.Value is INamedTypeSymbol { TypeKind: TypeKind.Interface } interfaceSymbol)
            {
                EquatableList<string> properties = new();

                foreach (IPropertySymbol interfaceProperty in interfaceSymbol
                    .GetMembers()
                    .OfType<IPropertySymbol>())
                {
                    string type = interfaceProperty.Type.ToDisplayString(SymbolDisplayFormat.FullyQualifiedFormat);

                    //Check if property has a setter
                    string setter = interfaceProperty.SetMethod is not null
                        ? "set; "
                        : string.Empty;

                    properties.Add($$"""
                            public {{type}} {{interfaceProperty.Name}} { get; {{setter}}}
                        """);
                }

                ret.Add(new InterfaceModel(interfaceSymbol.ToDisplayString(), properties));
            }
        }

        return ret;
    }

    private record ClassModel(string Name, string NameSpace, EquatableList<InterfaceModel> Interfaces);
    private record InterfaceModel(string FullyQualifiedName, EquatableList<string> Properties);

    private class EquatableList<T> : List<T>, IEquatable<EquatableList<T>>
    {
        public bool Equals(EquatableList<T>? other)
        {
            // If the other list is null or a different size, they're not equal
            if (other is null || Count != other.Count)
            {
                return false;
            }

            // Compare each pair of elements for equality
            for (int i = 0; i < Count; i++)
            {
                if (!EqualityComparer<T>.Default.Equals(this[i], other[i]))
                {
                    return false;
                }
            }

            // If we got this far, the lists are equal
            return true;
        }
        public override bool Equals(object obj)
        {
            return Equals(obj as EquatableList<T>);
        }
        public override int GetHashCode()
        {
            return this.Select(item => item?.GetHashCode() ?? 0).Aggregate((x, y) => x ^ y);
        }
        public static bool operator ==(EquatableList<T> list1, EquatableList<T> list2)
        {
            return ReferenceEquals(list1, list2)
                || list1 is not null && list2 is not null && list1.Equals(list2);
        }
        public static bool operator !=(EquatableList<T> list1, EquatableList<T> list2)
        {
            return !(list1 == list2);
        }
    }
}
```

## 重大更改：

* 目前无

## 待解决问题

本节跟踪其他杂项 TODO 项目：

**框架目标**：可能需要提及生成器是否有框架要求，例如它们必须以 netstandard2.0 或类似版本为目标。

**约定**：（参见上面[约定](#conventions)部分中的 TODO）。我们向用户建议哪些标准约定？

**功能检测**：展示如何创建依赖于特定目标框架功能的生成器，而不依赖于 TargetFramework 属性。
