C# 和 Visual Basic 编译器支持命令行上的 /errorlog:<file> 开关，以结构化的 JSON 格式记录所有诊断。

日志格式为 SARIF（静态分析结果交换格式）：
有关格式规范、JSON 架构和其他相关资源，请参阅 https://sarifweb.azurewebsites.net/。

## SARIF v2 格式

本节提供 C# 和 Visual Basic 编译器生成的 SARIF v2 错误日志文件内容的高级概述。SARIF v2 格式的更正式和详细的架构规范在 http://json.schemastore.org/sarif-2.1.0 中指定。

1. `schema` 和 `version` 信息：错误日志文件的前两行指定 SARIF 架构和版本信息。例如：

```json
"$schema": "http://json.schemastore.org/sarif-2.1.0",
"version": "2.1.0",
```

2. `runs` 信息：错误日志文件中的核心条目是 `runs` 部分，其中包含构建的单个运行条目。运行条目具有以下主要部分：
   1. `results` 部分：此部分包含结果条目数组，其中每个结果对应于报告的编译器或分析器 `Diagnostic` 的信息。更多详细信息请参阅[每个编译器或分析器 `Diagnostic` 实例的 `Result` 格式](#每个编译器或分析器-diagnostic-实例的-result-格式)。
   2. `properties` 部分：此部分包含与运行关联的自定义键值属性。当前发出以下属性：
      1. `analyzerExecutionTime`：所有分析器执行的执行时间（秒），如 `/reportAnalyzer` 编译器开关所报告。
   3. `tools` 部分：此部分包含有关编译器构建和版本的信息。此外，它包含一个 `rules` 数组，其中每个规则条目对应于每个报告的分析器诊断的元数据或 `DiagnosticDescriptor` 信息。更多详细信息请参阅[每个分析器支持的 `DiagnosticDescriptor` 实例的 `Rule` 格式](#每个分析器支持的-diagnosticdescriptor-实例的-rule-格式)
   4. `invocations` 部分：此部分包含有关编译器调用/运行的规则的最终用户指定配置的信息。仅当至少一个规则通过选项对编译的部分或全部具有一个或多个严重性配置覆盖时，我们才发出此部分。更多详细信息请参阅[每个规则严重性覆盖的 `Rule Configuration Override` 格式](#每个规则严重性覆盖的-rule-configuration-override-格式)。
   5. `columnKind` 部分：此部分包含有关工具测量列的单位的信息。C# 和 Visual Basic 编译器使用 utf16 代码单元。

示例 `runs` 部分，删除了 `results`、`rules` 和 `ruleConfigurationOverrides` 部分：
```json
"runs": [
{
  "results": [
  ],
  "properties": {
    "analyzerExecutionTime": "x.xxx"
  },
  "tool": {
    "driver": {
      "name": "Microsoft (R) Visual C# Compiler",
      "version": "4.4.0-dev (<developer build>)",
      "dottedQuadFileVersion": "42.42.42.42",
      "semanticVersion": "42.42.42",
      "language": "en-US",
      "rules": [
      ]
    }
  },
  "invocations": [
    {
      "executionSuccessful": true,
      "ruleConfigurationOverrides": [
      ]
    }
  ],
  "columnKind": "utf16CodeUnits"
}
]
```

### 每个编译器或分析器 `Diagnostic` 实例的 `Result` 格式

results 部分包含结果条目数组，其中每个结果对应于报告的编译器或分析器 `Diagnostic` 的信息。它包含以下数据：
1. `ruleId`：诊断的规则 ID 或诊断 ID。
2. `ruleIndex`：诊断的基础 `DiagnosticDescriptor` 的 `rules` 部分的索引。更多详细信息请参阅[每个分析器支持的 `DiagnosticDescriptor` 实例的 `Rule` 格式](#每个分析器支持的-diagnosticdescriptor-实例的-rule-格式)。
3. `level`：诊断的严重性级别，如 `error`、`warning`、`note` 等。
4. `message`：诊断的面向用户的消息。
5. `suppressions`：对于使用 pragma 指令、SuppressMessageAttribute 或通过 DiagnosticSuppressor 抑制的每个诊断实例，结果条目包含具有以下数据的 `suppressions` 部分：
   1. `kind` 信息：C# 和 Visual Basic 编译器支持单一 `inSource` 抑制类型。
   2. `justification` 信息：与抑制相关的理由。目前，此字段仅为基于 `SuppressMessageAttribute` 的抑制填充，具有非空理由参数。
   3. `suppressionType` 属性，具有以下三个值之一：`Pramga Directive`、`SuppressMessageAttribute` 或 `DiagnosticSuppressor`。此数据有助于分析代码库中首选的源内抑制机制。
6. `locations`：与诊断关联的一个或多个位置。
7. `properties`：与诊断关联的一个或多个自定义键值字符串对。

示例 `result` 条目：
```json
{
  "ruleId": "CA1822",
  "ruleIndex": 97,
  "level": "warning",
  "message": {
    "text": "Member 'M2' does not access instance data and can be marked as static"
  },
  "suppressions": [
  {
    "kind": "inSource",
    "justification": "<Pending>",
    "properties": {
      "suppressionType": "SuppressMessageAttribute"
    }
  }
  ],
  "locations": [
  {
    "physicalLocation": {
      "artifactLocation": {
        "uri": "file:///C:/source/repos/ClassLibrary1/Class1.cs"
      },
      "region": {
        "startLine": 13,
        "startColumn": 10,
        "endLine": 13,
        "endColumn": 12
      }
    }
  }
  ],
  "properties": {
    "warningLevel": 1
  }
}
```

### 每个分析器支持的 `DiagnosticDescriptor` 实例的 `Rule` 格式

`rules` 部分包含与每个分析器支持的描述符的元数据或 `DiagnosticDescriptor` 信息对应的规则条目。我们报告提供给编译的所有分析器的 `DiagnosticDescriptor` 信息，无论它们是否在整个编译、部分编译上执行，还是在整个编译中被禁用。`rule` 条目包含以下数据：
1. `id`：与描述符关联的规则 ID 或诊断 ID。
2. `shortDescription`：与描述符关联的面向用户的简短描述或 `Title`。
3. `fullDescription`：与描述符关联的面向用户的完整描述或 `Description`。
4. `defaultConfiguration`：为描述符报告的诊断的默认严重性级别，如 `error`、`warning`、`note` 等。
5. `helpUri`：与描述符关联的帮助信息的帮助 URI。
6. `properties`：与描述符关联的一个或多个自定义属性。它包括以下内容：
   1. `category`：与描述符关联的 `Category`，如 `Design`、`Performance`、`Security` 等。
   2. `isEverSuppressed` 和 `suppressionKinds`：如果规则通过选项具有源抑制或在编译的部分或全部中被禁用，则规则元数据包含特殊标志 `isEverSuppressed = true` 和数组 `suppressionKinds`，其中包含以下抑制类型之一或两者：
      1. `inSource` 抑制类型，用于一个或多个通过 pragma 指令、SuppressMessageAttribute 或 DiagnosticSuppressor 抑制的报告诊断。
      2. `external` 抑制类型，用于在整个编译中禁用的诊断 ID（通过全局选项如 /nowarn、ruleset、globalconfig 等）或在编译中某些文件或文件夹中禁用的诊断 ID（通过 editorconfig 选项）。
   3. `executionTimeInSeconds`：与分析器关联的执行时间（秒），如 `/reportAnalyzer` 编译器开关所报告。小于 `0.001` 秒的值报告为 `<0.001`。
   4. `executionTimeInPercentage`：与分析器关联的执行时间百分比，如 `/reportAnalyzer` 编译器开关所报告。小于 `1` 的值报告为 `<1`。
   5. `tags`：与描述符关联的一个或多个 `CustomTags` 数组。
  
示例 `rule` 条目：
```json
{
  "id": "CA1001",
  "shortDescription": {
    "text": "Types that own disposable fields should be disposable"
  },
  "fullDescription": {
    "text": "A class declares and implements an instance field that is a System.IDisposable type, and the class does not implement IDisposable. A class that declares an IDisposable field indirectly owns an unmanaged resource and should implement the IDisposable interface."
  },
  "defaultConfiguration": {
    "level": "note"
  },
  "helpUri": "https://docs.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1001",
  "properties": {
    "category": "Design",
    "isEverSuppressed": "true",
    "suppressionKinds": [
      "external"
    ],
    "executionTimeInSeconds": "x.xxx",
    "executionTimeInPercentage": "xx",
    "tags": [
      "PortedFromFxCop",
      "Telemetry",
      "EnabledRuleInAggressiveMode"
    ]
  }
}
```

### 每个规则严重性覆盖的 `Rule Configuration Override` 格式

`ruleConfigurationOverrides` 部分包含条目数组，每个条目对应于通过选项（即 editorconfig、globalconfig、命令行选项、ruleset 等）对编译的全部或部分进行的每个规则严重性配置覆盖。每个规则配置覆盖条目包含以下信息：
1. `descriptor`：包含映射回 `rules` 部分中相关 `descriptor` 或 `rule` 的信息的属性。目前，它包含以下子属性：
   1. `id`：规则或诊断的规则 ID 字符串。
   2. `index`：规则映射的 `rules` 数组的从零开始的整数索引值。
2. `configuration`：规则的有效覆盖严重性的配置属性。目前，它可能包含以下子属性之一：
   1. `enabled`：布尔属性，指示规则是否已通过配置覆盖条目显式禁用或启用。
   1. `level`：规则的有效严重性属性，如 `error`、`warning` 或 `note`。请注意，对于使用配置覆盖禁用的规则，不发出此属性，我们改为发出 `enabled: false`。

请注意，如果规则配置为在编译的不同部分以不同的严重性触发，我们可以为同一描述符有多个 `ruleConfigurationOverride` 条目。例如，基于 editorconfig 的严重性配置可以为项目中不同文件或文件夹中的同一规则 ID 指定不同的有效严重性。

示例 `ruleConfigurationOverride` 条目：
```json
{
  "descriptor": {
    "id": "CA1000",
    "index": 0
  },
  "configuration": {
    "level": "error"
  }
},
{
  "descriptor": {
    "id": "CA1000",
    "index": 0
  },
  "configuration": {
    "enabled": false
  }
},
{
  "descriptor": {
    "id": "CA1001",
    "index": 1
  },
  "configuration": {
    "level": "warning"
  }
}
```
