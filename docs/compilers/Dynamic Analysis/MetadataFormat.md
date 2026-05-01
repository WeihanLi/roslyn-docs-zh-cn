# 动态分析元数据格式规范 (v 0.2)

### 概述
该格式基于 ECMA-335 Partition II 元数据标准和 [Portable PDB 格式](https://github.com/dotnet/corefx/blob/main/src/System.Reflection.Metadata/specs/PortablePdb-Metadata.md) 中定义的概念。

## 元数据布局

动态分析元数据 blob 的物理布局以 [Header](#Header) 开始，后跟 [Tables](#Tables)，再后跟 [Heaps](#Heaps)。以下各节定义了这三个部分的布局。

当存储在托管 PE 文件中时，动态分析元数据 blob 作为名为 ```<DynamicAnalysisData>``` 的清单资源嵌入（参见 ECMA-335 §6.2.2 和 §22.24）。

除非另有说明，否则所有二进制值都以小端格式存储。

本文档使用术语 _（无）符号压缩整数_ 来表示 ECMA §23.2 中定义的（无）符号 29 位整数的编码。

## Header

| 偏移量  | 大小 | 字段          | 描述                                                    |
|:--------|:-----|:---------------|----------------------------------------------------------------|
| 0       | 4    | Signature      | 0x44 0x41 0x4D 0x44 (ASCII 字符串: "DAMD") |
| 4       | 1    | MajorVersion   | 格式的主版本号 (0) |
| 5       | 1    | MinorVersion   | 格式的次版本号 (2) |
| 6       | 4*T  | TableRowCounts | 元数据中每个表的行数（编码为 uint32） |
| 6 + T*4 | 4*H  | HeapSizes      | 每个堆的大小（以字节为单位，编码为 uint32） |

此版本格式中的表数量为 T = 2。这些表是 Document Table 和 Method Table，它们的大小按此顺序存储。任何表的行数都不能超过 0x1000000 行。

此版本格式中的堆数量为 H = 2。这些堆是 GUID Heap 和 Blob Heap，它们的大小按此顺序存储。任何堆的大小都不能超过 2^19 字节（0.5 GB）。

## <a name="Tables"></a>Tables

存储在表中的实体如果在隐含表的上下文中使用，则通过 _row id_ 引用。表的第一行的 row id 为 1。如果上下文没有隐含表，则实体通过其 _token_ 引用——一个 32 位无符号整数，它组合了表的 id（在最高 8 位中）和该表中实体的 row id（在最低 24 位中）。

### <a name="DocumentTable"></a>Document Table: 0x01

Document 表有以下列：
* _Name_（[document name blob](#DocumentNameBlob) 的 Blob 堆索引）
* _HashAlgorithm_（Guid 堆索引）
* _Hash_（Blob 堆索引）

### <a name="MethodTable"></a>Method Table: 0x02

Method 表有以下列：
* _Spans_（[span blob](#SpanBlob) 的 Blob 堆索引）

## <a name="Heaps"></a>Heaps

### GUID

GUID 堆的编码与 ECMA-335 §24.2.5 中定义的 ECMA #GUID 堆的编码相同。

存储在 GUID 堆中的值通过其 _index_ 引用。存储在堆中的第一个值的索引为 1，存储在堆中的第二个值的索引为 2，依此类推。

### Blob

Blob 堆的编码与 ECMA-335 §24.2.4 中定义的 ECMA #Blob 堆的编码相同。

存储在 Blob 堆中的值通过其在堆中的 _offset_（堆的开始和编码值的第一个字节之间的距离）引用。堆的第一个值的偏移量为 0，大小为 1B，编码值为 0x00（它表示空 blob）。

### <a name="SpansBlob"></a>Spans Blob

_Span_ 是整数四元组和文档引用：

* Start Line
* Start Column
* End Line
* End Column
* Document

值必须满足以下约束

* Start Line 在范围 [0, 0x20000000) 内
* End Line 在范围 [0, 0x20000000) 内
* Start Column 在范围 [0, 0x10000) 内
* End Column 在范围 [0, 0x10000) 内
* End Line 大于或等于 Start Line。
* 如果 Start Line 等于 End Line，则 End Column 大于 Start Column。

_Spans blob_ 具有以下结构：

    Blob ::= header span-record (span-record | document-record)*

#### header

| 组件          | 存储的值                  | 整数表示 |
|:-------------------|:------------------------------|:-----------------------|
| _InitialDocument_  | Document row id               | unsigned compressed    |

#### span-record

| 组件      | 存储的值                                            | 整数表示                      |
|:---------------|:--------------------------------------------------------|:--------------------------------------------|
| _ΔLines_       | _EndLine_ - _StartLine_                                 | unsigned compressed                         |
| _ΔColumns_     | _EndColumn_ - _StartColumn_                             | _ΔLines_ = 0: unsigned compressed, non-zero |
|                |                                                         | _ΔLines_ > 0: signed compressed             |
| _δStartLine_   | 如果这是第一个 _span-record_ 则为 _StartLine_          | unsigned compressed                         |
|                | 否则为 _StartLine_ - _PreviousSpan_._StartLine_      | signed compressed                           |
| _δStartColumn_ | 如果这是第一个 _span-record_ 则为 _StartColumn_        | unsigned compressed                         |
|                | 否则为 _StartColumn_ - _PreviousSpan_._StartColumn_  | signed compressed                           |

其中 _PreviousSpan_ 是前一个 _span-record_ 中编码的 span。

#### document-record
| 组件    | 存储的值                       | 整数表示         |
|:-------------|:-----------------------------------|:-------------------------------|
| _ΔLines_     | 0                                  | unsigned compressed            |
| _ΔColumns_   | 0                                  | unsigned compressed            |
| _Document_   | Document row id                    | unsigned compressed            |

每个 _span-record_ 代表单个 _Span_。解码 blob 时，_Span_ 的 _Document_ 属性由最近的前一个 _document-record_ 确定，如果没有前一个 _document-record_，则由 _InitialDocument_ 确定。

_Span_ 的 _Start Line_、_Start Column_、_End Line_ 和 _End Column_ 的值根据前一个 Span（如果有）的值和记录中存储的数据计算得出。

- - -
**注意** 此编码类似于 Portable PDB 格式中 [sequence points blob](https://github.com/dotnet/corefx/blob/main/src/System.Reflection.Metadata/specs/PortablePdb-Metadata.md#SequencePointsBlob) 的编码。
- - -

### <a name="DocumentNameBlob"></a>Document Name Blob

_Document name blob_ 是一个序列：

    Blob ::= separator part+

其中

* _separator_ 是 UTF8 编码的字符，或字节 0 表示空分隔符。
* _part_ 是 Blob 堆中的压缩整数，其中部分以 UTF8 编码存储（0 表示空字符串）。

文档名称是由 _separator_ 分隔的 _parts_ 的连接。

- - -
**注意** 此编码与 Portable PDB 格式中 [document name blob](https://github.com/dotnet/corefx/blob/main/src/System.Reflection.Metadata/specs/PortablePdb-Metadata.md#DocumentNameBlob) 的编码相同。
- - -
