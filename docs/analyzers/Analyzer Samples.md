**示例位置：**

用于演示不同分析场景推荐实现模型的示例分析器已添加到 [Samples.sln](https://github.com/dotnet/roslyn-sdk/tree/main/Samples.sln)。

**描述：**

分析器根据执行的分析类型大致分为以下三类：
  1. 无状态分析器：报告关于特定代码单元（如符号、语法节点、代码块、编译等）的诊断的分析器。大多数分析器属于此类别。
     这些分析器注册一个或多个操作，这些操作：
     1. 不需要在分析器操作之间维护状态，且
     2. 独立于各个分析器操作的执行顺序。
  2. 有状态分析器：报告关于特定代码单元的诊断的分析器，但是在封闭代码单元（如代码块或编译）的上下文中。这些代表了稍微复杂一些的分析器，需要更强大和更广泛的分析，因此需要仔细设计以实现高效的分析器执行而不会发生内存泄漏。
     这些分析器需要以下至少一种状态操作进行分析：
     1. 访问封闭代码单元（如编译或代码块）的不可变状态对象。例如，访问编译中定义的某些众所周知的类型。
     2. 对封闭代码单元执行分析，在封闭代码单元的开始操作中定义和初始化可变状态，中间操作访问和/或更新此状态，以及在结束操作中报告各个代码单元的诊断。
  3. 附加文件分析器：从项目中包含的非源文本文件读取数据的分析器。
		
**内容：**
	
提供了以下示例分析器及简单的单元测试：
  1. [无状态分析器](https://github.com/dotnet/roslyn-sdk/tree/master/samples/CSharp/Analyzers/Analyzers.Implementation/StatelessAnalyzers)：
     1. SymbolAnalyzer：用于报告符号诊断的分析器。
     2. SyntaxNodeAnalyzer：用于报告语法节点诊断的分析器。
     3. CodeBlockAnalyzer：用于报告代码块诊断的分析器。
     4. CompilationAnalyzer：用于报告编译诊断的分析器。
     5. SyntaxTreeAnalyzer：用于报告语法树诊断的分析器。
     6. SemanticModelAnalyzer：用于报告需要某些语义分析的语法树诊断的分析器。
  2. [有状态分析器](https://github.com/dotnet/roslyn-sdk/tree/master/samples/CSharp/Analyzers/Analyzers.Implementation/StatefulAnalyzers)：
     1. CodeBlockStartedAnalyzer：演示代码块范围分析的分析器。
     2. CompilationStartedAnalyzer：演示编译内分析的分析器，例如依赖于某些众所周知的符号的分析。
     3. CompilationStartedAnalyzerWithCompilationWideAnalysis：演示编译范围分析的分析器。
  3. [附加文件分析器](https://github.com/dotnet/roslyn-sdk/tree/master/samples/CSharp/Analyzers/Analyzers.Implementation/AdditionalFileAnalyzers)：
     1. SimpleAdditionalFileAnalyzer：演示逐行读取附加文件并在分析中使用数据。
     2. XmlAdditionalFileAnalyzer：演示将附加文件写入 `Stream`，以便可以作为结构化文档（在本例中为 XML）读回。
