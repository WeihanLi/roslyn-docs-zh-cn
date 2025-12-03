简介
============
C# 和 Visual Basic 编译器在命令行上支持 `/reportanalyzer` 开关，用于报告额外的分析器信息，例如执行时间。

输出包含执行分析器所花费的总挂钟时间和每个分析器的相对执行时间。
请注意，经过时间可能小于分析器执行时间，因为分析器可以并发运行。应仅将此数据用于比较分析，以识别任何比其他分析器花费显著更多时间的异常分析器。

MSBuild 命令
=============
使用以下 msbuild 命令行获取分析器性能报告：

```
msbuild.exe /v:d /p:reportanalyzer=true <%project_file%>
```

输出格式
=============

```
Total analyzer execution time: XYZ seconds.
NOTE: Elapsed time may be less than analyzer execution time because analyzers can run concurrently.
```
时间 (秒)  | % |  分析器
----------|---|----------------------
xyz1      | X |  分析器程序集标识
xyz2      | Y |    -    DiagnosticAnalyzer1
xyz3      | Z |    -    DiagnosticAnalyzer2


示例
=============

以下是使用分析器构建的项目的示例输出：

-------------------------------------------------------------------------------------------------------
```
Total analyzer execution time: 0.503 seconds.
NOTE: Elapsed time may be less than analyzer execution time because analyzers can run concurrently.

Time (s)    %   Analyzer
   0.242   48   Roslyn.Diagnostics.CSharp.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.240   47      Roslyn.Diagnostics.Analyzers.CSharpCodeActionCreateAnalyzer
   0.002   <1      Roslyn.Diagnostics.Analyzers.CSharp.CSharpSpecializedEnumerableCreationAnalyzer
  <0.001   <1      Roslyn.Diagnostics.Analyzers.CSharp.CSharpDiagnosticDescriptorAccessAnalyzer
  <0.001   <1      Roslyn.Diagnostics.Analyzers.CSharpSymbolDeclaredEventAnalyzer

   0.137   27   System.Runtime.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.045    8      System.Runtime.Analyzers.AttributeStringLiteralsShouldParseCorrectlyAnalyzer
   0.024    4      System.Runtime.Analyzers.SpecifyStringComparisonAnalyzer
   0.022    4      System.Runtime.Analyzers.SpecifyIFormatProviderAnalyzer
   0.020    3      System.Runtime.Analyzers.ProvideCorrectArgumentsToFormattingMethodsAnalyzer
   0.009    1      System.Runtime.Analyzers.CallGCSuppressFinalizeCorrectlyAnalyzer
   0.007    1      System.Runtime.Analyzers.DisposableTypesShouldDeclareFinalizerAnalyzer
   0.003   <1      System.Runtime.Analyzers.NormalizeStringsToUppercaseAnalyzer
   0.003   <1      System.Runtime.Analyzers.InstantiateArgumentExceptionsCorrectlyAnalyzer
   0.002   <1      System.Runtime.Analyzers.TestForNaNCorrectlyAnalyzer
   0.001   <1      System.Runtime.Analyzers.DoNotUseEnumerableMethodsOnIndexableCollectionsInsteadUseTheCollectionDirectlyAnalyzer
  <0.001   <1      System.Runtime.Analyzers.SpecifyCultureInfoAnalyzer
  <0.001   <1      System.Runtime.Analyzers.DoNotLockOnObjectsWithWeakIdentityAnalyzer
  <0.001   <1      System.Runtime.Analyzers.TestForEmptyStringsUsingStringLengthAnalyzer
  <0.001   <1      System.Runtime.Analyzers.AvoidUnsealedAttributesAnalyzer

   0.085   16   Roslyn.Diagnostics.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.085   16      Roslyn.Diagnostics.Analyzers.DeclarePublicAPIAnalyzer

   0.023    4   System.Runtime.CSharp.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.014    2      System.Runtime.Analyzers.CSharpAvoidZeroLengthArrayAllocationsAnalyzer
   0.008    1      System.Runtime.Analyzers.CSharpInitializeStaticFieldsInlineAnalyzer
   0.001   <1      System.Runtime.Analyzers.CSharpDoNotRaiseReservedExceptionTypesAnalyzer
  <0.001   <1      System.Runtime.Analyzers.CSharpUseOrdinalStringComparisonAnalyzer
  <0.001   <1      System.Runtime.Analyzers.CSharpDoNotUseTimersThatPreventPowerStateChangesAnalyzer
  <0.001   <1      System.Runtime.Analyzers.CSharpDisposeMethodsShouldCallBaseClassDisposeAnalyzer

   0.006    1   System.Runtime.InteropServices.CSharp.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.006    1      System.Runtime.InteropServices.Analyzers.CSharpAlwaysConsumeTheValueReturnedByMethodsMarkedWithPreserveSigAttributeAnalyzer
  <0.001   <1      System.Runtime.InteropServices.Analyzers.CSharpMarkBooleanPInvokeArgumentsWithMarshalAsAnalyzer
  <0.001   <1      System.Runtime.InteropServices.Analyzers.CSharpUseManagedEquivalentsOfWin32ApiAnalyzer

   0.004   <1   System.Runtime.InteropServices.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.004   <1      System.Runtime.InteropServices.Analyzers.PInvokeDiagnosticAnalyzer

   0.004   <1   System.Collections.Immutable.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.004   <1      System.Collections.Immutable.Analyzers.DoNotCallToImmutableArrayOnAnImmutableArrayValueAnalyzer

   0.001   <1   System.Threading.Tasks.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.001   <1      System.Threading.Tasks.Analyzers.DoNotCreateTasksWithoutPassingATaskSchedulerAnalyzer

   0.001   <1   XmlDocumentationComments.CSharp.Analyzers, Version=1.2.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35
   0.001   <1      XmlDocumentationComments.Analyzers.CSharpAvoidUsingCrefTagsWithAPrefixAnalyzer
```
