# Roslyn 增量解析：深度解析

本文档介绍增量解析在 Roslyn C# 编译器中的工作原理。本文面向从事 Roslyn 解析器工作或与其打交道的开发者，并假设读者熟悉基本的编译器概念。

如需更多背景知识，Neal Gafter 的 [Toy-Incremental-Parser](https://github.com/gafter/Toy-Incremental-Parser/blob/main/README.md) 提供了一个类似技术的教学参考实现。Roslyn 与之有许多共同之处，但有其自己的生产实现。

## 为什么增量解析至关重要

在 IDE 中，用户期望在输入时获得即时反馈。每次按键都可能改变语法树，而智能感知、错误波浪线和括号匹配等功能都依赖于最新的解析结果。对于小文件，从头重新解析足够快，几乎感知不到。但对于一个包含 100,000 行代码的测试文件或大型 API 客户端来说，每次按键都重新解析整个文件将引入明显的延迟。

增量解析通过尽可能重用之前的解析树来解决这一问题。当用户在一个方法中键入一个字符时，我们无需重新解析其前后的数千个方法。

## 基础：Roslyn 解析器

在深入增量解析之前，我们需要了解 Roslyn 如何解析 C# 的一些基础方面。

### 基本上下文无关的递归下降

Roslyn 使用手写的递归下降解析器。该解析器*基本上*是上下文无关的，这意味着 `ClassDeclaration`、`MethodDeclaration`、`Statement` 和 `Expression` 等语言产生式直接对应 [`LanguageParser.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs) 中的解析函数。

"基本上"这个限定词很重要。C# 有一些上下文相关的结构；例如，`await` 只有在 `async` 方法内部才被视为关键字。解析器通过标志位跟踪这些上下文，正如我们将看到的，这对增量解析有影响。但整体设计努力最小化上下文敏感性，以最大化增量重用的潜力。

### 绿色节点：无位置、无父节点

关于"绿色节点"的完整详细描述，请参阅 [Red-Green Trees](./Red-Green Trees.md)。

Roslyn 使用"红/绿树"模式。解析器生成**绿色节点**，这些不可变节点只存储其*种类*和*子节点*。至关重要的是，绿色节点**不**存储：

- 绝对文本位置（spans）
- 父节点指针

相反，绿色节点只存储其*宽度*（它所跨越的字符数）。这一设计选择是增量解析的基础：由于绿色节点不编码绝对位置，即使文件前面的编辑改变了节点的位置，它们仍可被重用。旧树中位置 10,000 的方法声明，在新树中位置 10,050 处仍可被重用，因为绿色节点本身是完全相同的。

更重要的是，**重用意味着字面上的对象重用**。当我们说一个节点被"重用"时，意味着新树指向内存中完全相同的对象。一次解析产生的巨大子树可以*原样*用于下一棵树，无需任何复制。唯一的新分配是从根节点到编辑位置的路径上的父节点。

这就是增量解析如此高效的原因。一棵语法树可能有几十兆字节大小。如果没有增量解析，每次按键都会分配几十兆字节的内存。有了增量解析，典型的编辑只分配数字节：少量新的父节点和一些指针。其代价与编辑的*影响*（通常极小）相称，而非与文件的大小相称。

**红色节点**是惰性构建的包装器，基于绿色节点提供带有 span 和父节点导航功能的熟悉 API。它们在用户代码遍历树时按需构建。

### 完全保真的具体语法树

关于"完全保真"的完整详细描述，请参阅 [Red-Green Trees](./Red-Green Trees.md)。

Roslyn 生成*完全保真*的语法树。源文件中的每个字符在树中都有其对应的表示，包括空白符、注释，甚至语法错误。如果将所有 token（包括其 trivia）按顺序拼接，可以得到完全原始的源文本，不多不少。

这一属性对增量解析至关重要。由于树代表了*完整的*源文本，节点和 token 与被解析的文本段字面上是同构的。这意味着它们可以安全地被用作原始文本的代理。只有当某些原因导致我们无法重用旧节点或 token 时，才需要参考新文本。

### 列表模式

解析树是层次化的，节点包含子节点列表：

- `CompilationUnit` 包含成员列表（命名空间、类型等）
- `NamespaceDeclaration` 包含成员列表
- `ClassDeclaration` 包含类型成员列表（方法、属性、字段等）
- `BlockStatement` 包含语句列表

这种"父节点包含子节点列表"的重复模式非常重要，因为它创建了增量重用的自然边界。

## 增量解析的工作原理

### 增量解析函数

普通（非增量）解析是从文本到语法树的函数：

```
Parse: Text → SyntaxTree
```

增量解析是一个函数，它接受*新*文本、*旧*树以及将旧文本转换为新文本的*变更*：

```
IncrementalParse: (NewText, OldTree, TextChange) → NewSyntaxTree
```

`TextChange` 描述了旧文本的哪个区域被替换为什么内容。有了这些信息，增量解析器可以确定旧树的哪些部分仍然有效并可被重用。

### 通过 Blender 实现 Token 重用

增量解析的核心是一个名为 **blender**（[`Blender.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.cs)、[`Blender.Reader.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.Reader.cs)）的组件。blender 的工作是向解析器提供 token，这些 token 要么来自旧树（安全时），要么来自词法器（必要时）。

可以将 blender 理解为维护一个进入旧树的**游标**，跟踪*新*文本中哪个位置对应于*旧*树中的哪个位置。随着解析器消耗 token，blender 推进此游标。

#### 位置同步与变更偏移量

blender 跟踪一个 [`_changeDelta`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.Reader.cs#L30)，它是当前位置之前旧文本和新文本之间长度的累积差值。在编辑位置之前，旧文本和新文本中的位置是相同的，因此 `_changeDelta` 为零。编辑之后，旧树中的位置需要通过此偏移量进行调整，以找到新文本中的对应位置。

例如，如果用户在位置 100 处将 `1` 替换为 `true`，偏移量为 +3（四个字符替换一个字符）。位置 0 到 99 保持不变，但旧树中的位置 100 现在对应新文本中的位置 103。

这一记录方式让 blender 能够判断游标何时"同步"，即新文本中的当前位置（经过调整后）与旧树中的某个 token 或节点边界对齐。

#### 解析器如何了解其位置

尽管绿色节点不存储绝对位置，解析器始终确切地知道自己在新文本中的位置。它从位置 0 开始，累积每个处理过的 token 或节点的宽度。在任何时刻，解析器都知道其精确位置。

将此位置映射回旧文本非常简单：如果解析器在编辑位置之前，新旧文本中的位置相同；如果在编辑位置之后，只需应用偏移量即可映射到旧文本中的对应位置。

#### Blender 同步的工作原理

解析器根据新文本中的位置向 blender 请求数据。blender 在内部将其转换为旧文本中的对应位置。

blender 也始终知道自己在旧树中的位置。它从位置 0 开始，每次移过一个节点或 token 时，都知道移过的宽度并相应更新其位置。

这就是 blender 判断是否可以返回旧 token 或节点的方式：如果混合使其到达某个节点或 token，其起始位置与映射后的旧文本位置对齐，则可以返回该节点或 token；否则，位置不同步，blender 必须回退到对新文本进行词法分析。

词法分析持续进行，直到 blender 到达旧文本位置再次与节点或 token 边界对齐的点。此时，重用可以恢复。

对于典型的编辑，这种重新同步几乎在编辑区域之后立即发生。然而，影响较大的编辑（如插入 `/*`）可能会在重新同步之前引起更多波动。

#### Token 可以被重用的条件

当解析器请求下一个 token 时，blender 检查：

1. 游标是否与旧树中的某个 token 同步？
2. 该 token 是否完全位于编辑区域之外？
3. 该 token 是否通过可重用性检查（详见下文）？

如果所有条件都满足，blender 直接返回旧 token，无需任何词法分析。这既节省了 CPU 时间（跳过词法器逻辑），也节省了内存（重用现有的 token 对象）。

#### 混合与分解

"混合"（有时称为"分解"）描述了旧树被拆解以供重用的方式。blender 维护一个来自旧树的项目队列。队列前面的节点被逐步分解为其子节点，直到 token 出现在队列前面。

此过程如下：

1. 从旧树的根节点开始
2. 当一个节点与编辑相交（或因其他原因无法重用）时，**分解**它：将其从队列中移除，并将其子节点压入队列
3. 继续直到 token 位于队列前面
4. 根据请求向解析器返回 token

这种惰性分解意味着我们只需分解实际需要检查的部分。完全位于编辑之前或之后的巨大子树保持完整。

此过程的输出捕获在一个 [`BlendedNode`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/BlendedNode.cs) 结构中，该结构携带重用的节点、重用的 token 或新词法分析的 token，并返回给解析器。

### 关键点处的节点重用

Token 重用很有价值，但增量解析的真正威力来自于**节点重用**。解析器不必逐个 token 地重新解析整个方法声明，而是可以从旧树中直接获取整个 `MethodDeclarationSyntax` 节点并整体重用。

这发生在解析器的**关键点**，具体是在解析高层结构列表的循环中：

- **编译单元成员**：命名空间、顶层类型、全局语句
- **类型成员**：方法、属性、字段、嵌套类型等
- **语句**：方法体和代码块的内容

选择这些点不仅因为它们代表自然的列表边界，还因为它们不受前向查找问题的影响（见下文[为什么不重用表达式？](#为什么不重用表达式)的解释）。

典型示例包括：
- [`TryReuseStatement`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L8334)（从 `ParseStatementCore` 调用）
- [`CanReuseMemberDeclaration`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L2442)（解析类型成员时调用）

在每个这些点，解析器检查 [`IsIncrementalAndFactoryContextMatches`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L14289)，这是一个属性，用于验证我们正在进行增量解析，并且解析器的当前上下文与候选节点最初解析时的上下文匹配。

### 为什么不重用表达式？

你可能注意到表达式明显不在可重用结构的列表中。这是有意为之，与解析器中的前向查找有关。

C# 中的表达式解析涉及大量前向查找、后向查找和上下文敏感性：

- `<` 是泛型参数列表的开始还是小于运算符？
- `(` 是类型转换、括号表达式、元组、lambda、解构，还是未来可能更多的含义？
- 优先级如何影响分组？

由于这种复杂性，确定表达式节点在编辑后是否可以安全重用出人意料地困难。*之后*的编辑可能会追溯性地改变该表达式应该如何被解析。Roslyn 采取了保守的方式，而非构建复杂（且容易出错）的逻辑来检测这些情况：表达式总是被重新解析。

相比之下，关键重用点（成员、语句）被明确选中，正是因为它们*不*受这些前向查找问题的影响。如果一个语句之前没有错误地解析（因此可以被重用），那么发生在该语句之后的编辑不会改变它应该被解析的方式。验证这一点需要理解语法和解析器实现，然后非正式地验证这些结构的终止点之外不需要前向查找。

重新解析表达式是一个实际的权衡。表达式通常很小，因此重新解析代价很低。真正的收益来自于重用语句和成员声明，它们可以任意大。（边缘情况请参见[注意事项](#注意事项)，其中此假设可能失效。）

## 工作示例

### 典型的编辑

考虑一个正在编辑的大型类：

```csharp
class HugeClass
{
    // ... Methods 0-499 ...

    void Method500()
    {
        // ... Statements 0-99 ...
        
        var x = 1;  // ← 用户将"1"改为"true"
        
        // ... Statements 101-200 ...
    }

    // ... Methods 501-1000 ...
}
```

用户将 `1`（1 个字符）替换为 `true`（4 个字符），因此变更偏移量为 +3。增量重新解析的过程如下：

1. **初始遍历**：blender 从根节点向下遍历，寻找编辑位置。它看到 `CompilationUnitSyntax` 包含编辑，因此将其分解为子节点。

2. **重用未受影响的成员**：方法 0 到 499 与编辑不相交。解析器在循环遍历类型成员时，对每个成员调用 `CanReuseMemberDeclaration`，它们都被整体重用，无需重新解析其内容。

3. **分解受影响的方法**：方法 500 与编辑相交，无法重用。blender 将其分解为子节点（修饰符、返回类型、名称、参数、方法体）。

4. **重用未受影响的语句**：在方法 500 的方法体内，语句 0 到 99 与编辑不相交。解析器在循环遍历语句时，调用 `TryReuseStatement` 并全部重用它们。

5. **重新解析受影响的语句**：包含 `var x = 1` 的语句与编辑相交，逐个 token 重新解析。（此语句中编辑位置之前的 token 仍可从旧树中重用。）

6. **重用剩余语句**：语句 101 到 200 位于编辑之后。blender 根据 +3 偏移量进行调整，与旧树重新同步，这些语句被重用。

7. **重用剩余成员**：方法 501 到 1000 位于编辑之后。它们全部被重用，同样通过偏移量进行位置调整。

结果：在可能数千个节点和数万个 token 中，实际上只有少数几个被重新解析。新树几乎所有的绿色节点（内存中的实际对象）都与旧树共享。

### 影响较大的编辑

并非所有编辑都如此局部。考虑当有人在方法 250 中输入 `/*`，而方法 750 中恰好有匹配的 `*/` 时会发生什么：

```csharp
class HugeClass
{
    // ... Methods 0-249 ...

    void Method250() 
    { 
        /*  // ← 用户输入此内容

    // ... Methods 251-749（现在位于注释内！）...

    void Method750() 
    {
        var s = "*/";  // ← 此处关闭注释
        // ...
    }

    // ... Methods 751-1000 ...
}
```

在这种情况下，`/*` 将使其后的 token 无效。当词法器运行时，它将生成一个从方法 250 内部一直延伸到方法 750 的巨大多行注释 token。方法 251 到 749 现在*位于*该注释 token 内部，无法作为成员被重用。方法 750 中 `*/` 之后的右大括号 `}` 现在关闭的是方法 250 而非方法 750。

然而，系统仍然能够正确工作。blender 将跳过旧树的大片区域（这些区域现在被注释 token 消耗），在 `*/` 之后重新同步，并恢复正常的增量解析。方法 751 到 1000 仍可像以前一样被重用。

这说明了一个重要的观点：增量解析的代价与编辑的*影响*相称，而不仅仅是其大小。简单的 `1 → true` 编辑对语法的影响微乎其微，因此重用率极高。注释掉半个文件对语法的影响极大，因此代价更高。

对于典型的低影响编辑（绝大多数真实世界的输入），增量解析在*微秒*内完成，内存重用率接近 99.99%。影响较大的编辑代价更高，但即使在最坏的情况下（注释掉整个文件），代价也等同于完整重新解析的代价。增量解析绝不会比完整解析明显更差，而通常要好得多。

## 正确性约束

增量解析器是保守的。它只在*确定*节点仍然有效时才重用它们。几个检查强制执行这一点：

### 被跳过的 Token

被跳过的 token（解析器无法将其纳入有效结构但必须放入树中以保持完全保真不变量的 token）从不被重用。它们的存在表明解析器遇到了意外情况，而由于编辑，该情况可能已经改变。

以一个包含两个方法的文件为例，M1 包含被跳过的 token（由于语法错误），用户在 M2 中进行编辑：

```csharp
class Example
{
    void M1()
    {
        int x = #;  // ← 此处有被跳过的 token（#）
    }

    void M2()
    {
        var y = 1;  // ← 用户将"1"改为"true"
    }
}
```

尽管编辑完全位于 M2 内部，M1 也不符合重用条件，因为它包含被跳过的 token。在增量重新解析过程中，M1 将被分解并重新解析。（注意，M1 内部本身不包含被跳过 token 的子语句仍可被重用。）

在大多数情况下，重新解析 M1 将产生与之前相同的被跳过 token。然而，M2 中的编辑可能因前向查找而影响 M1 的解析。例如，如果 M1 的错误是由 M2 中的编辑以某种方式解决的某些原因引起的，则 M1 的重新解析可能会产生不同的（可能是有效的）树。

这种保守的方式确实意味着一些额外的重新解析和分配。然而，解析错误往往很少见，在任意树中只占极小的比例。因此，这只会稍微增加实践中的解析代价。

### 缺失的节点和 Token

缺失的节点和 token（在错误恢复期间，解析器期望某些内容但未找到时插入的）从不被重用。它们是解析过程的合成产物，不代表实际的源文本。其行为与被跳过的 token 实际上相同：我们不因同样的原因重用包含它们的节点。

### 其他诊断信息

从技术上讲，解析器可以生成被跳过和缺失 token 之外的其他类型诊断信息。这些情况很少见，我们正积极努力将它们从解析器移至绑定阶段（如 [`Parser.md`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/docs/compilers/Design/Parser.md) 中所讨论的）。无论如何，它们的处理方式相同：包含任何诊断信息的节点不被重用，将被进一步分解。

### 上下文敏感性：解析器标志

如前所述，C# 有一些上下文相关的解析。例如：

- `await` 在 `async` 方法中是关键字，在其他地方是标识符
- `field` 在属性访问器内部是关键字（用于 field 关键字特性），在其他地方是标识符

解析器通过存储在节点上的位标志跟踪这些上下文。在考虑重用节点时，解析器检查 `IsIncrementalAndFactoryContextMatches`，它验证当前解析上下文是否与该节点最初创建时的活跃上下文匹配。如果不匹配，节点将被分解并重新解析。

### 合成 Token

某些 token 由解析器而非词法器合成。典型的例子是 `>>`，它可能代表：

- 右移运算符
- 嵌套泛型中的两个右尖括号（`List<List<int>>`）

这些 token 不能简单地从旧树中重用，因为其解释取决于解析上下文。遇到时，需要重新检查。

## 注意事项

增量解析器针对预期的使用场景进行了优化：正在被开发者活跃编辑的真实 C# 文件。某些边缘情况可能导致性能下降。

### 巨大的表达式

表达式很小的假设并不总是成立。拥有包含百万个元素的数组初始化器或巨大的插值字符串的用户，在编辑该表达式时只能获得 token 重用，表达式内部不会发生节点重用。

这可能导致 CPU 和内存方面的大量波动。实际上，这种情况很少见，但并非没有。工具和性能分析可以帮助发现这些情况。一般建议是将巨大的表达式分解为较小的部分（例如，从较小的数组构建数组），以使增量解析更有效。

### 存在大量错误的文件

增量解析假设树的大部分是格式良好的。遍布语法错误的文件将有许多包含诊断信息或被跳过 token 的节点，这些节点无法被重用。在极端情况下（随机文本、意外以 C# 方式打开的二进制文件），几乎没有任何内容可以被重用。

这些场景远远超出正常范围。我们是为真实的 C# 文件进行优化，而非任意文本。

### 生成的文件

生成的文件（源代码生成器、T4 模板等）不是增量解析性能的关注点。用户不会实时编辑生成的文件，因此我们不会为它们接收增量编辑。对生成文件进行完整解析是可以接受的，因为这只在生成器运行时发生（通常不频繁）。

Razor 文件是一个值得注意的例外。尽管 Razor 使用源代码生成，但每当用户编辑时，生成的 C# 代码都会重新生成。这意味着 Razor 文件每次都会完整地重新解析，而不是增量解析。对于小型 Razor 文件，这不会引起注意。然而，随着用户编写更大的 Razor 页面，完整的重新解析可能成为性能瓶颈。

一个潜在的解决方向是让 Razor 避免单独重新解析完整的生成文件。它可以先执行一个非常快速的差异（线性比较新旧文本的匹配前缀和后缀，以检测第一个变更的开始和最后一个变更的结束），然后将该变更区域输入增量解析。由于大多数编辑影响生成输出的一小部分，这可以显著降低大型 Razor 文件的解析代价。

### 深层嵌套的结构

未命中关键重用点（语句、成员）的深层嵌套结构不会受益于节点重用。例如，没有大括号的数千个嵌套 `if` 语句，或深度嵌套的三元表达式。这在某种程度上是病态的，实际中很少见，但值得注意。

## 关键代码位置

对于想要探索实现的开发者：

### 核心文件

- [`LanguageParser.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs)：主递归下降解析器
- [`Blender.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.cs)：协调来自旧树或词法器的 token/节点供应
- [`Blender.Reader.cs`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.Reader.cs)：实现游标逻辑和重用检查的 `Reader` 结构体

### 关键方法和属性

此列表并不详尽，但可以对主要关键点有一个宏观了解：

- [`TryReuseStatement`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L8334)：解析语句时调用；尝试重用语句节点
- [`CanReuseMemberDeclaration`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L2442)：解析类型成员时调用
- [`IsIncrementalAndFactoryContextMatches`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/LanguageParser.cs#L14289)：通过上下文检查保护节点重用
- [`CanReuse`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.Reader.cs#L215)（在 `Blender.Reader` 中）：检查 token/节点可重用性的所有约束
- [`_changeDelta`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/src/Compilers/CSharp/Portable/Parser/Blender.Reader.cs#L30)：跟踪新旧文本之间的位置偏移

### 相关文档

- [`docs/compilers/Design/Parser.md`](https://github.com/dotnet/roslyn/blob/b2cfaaf967aaad26cd58e7b2cc3f2d9fcede96f4/docs/compilers/Design/Parser.md)：解析器的设计指南，包括关于诊断信息放置如何影响增量解析的说明
- [Red-Green Trees](./Red-Green%20Trees.md)：深入解析 Roslyn 的红/绿语法节点分离，重点介绍内部绿色部分
