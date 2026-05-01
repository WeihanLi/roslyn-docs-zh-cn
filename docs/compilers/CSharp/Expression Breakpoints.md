# 表达式断点

Roslyn C# 编译器具有基础设施，可以在表达式内（而不仅仅是在语句边界处）生成断点。支持断点的第一种表达式形式是 *switch 表达式*。因为断点位置对应源代码中的序列点，这是通过生成额外的序列点来实现的。序列点被限制为只出现在代码中求值栈为空的位置，因此任何支持断点的表达式都使用 `BoundSpillSequence` 进行转换。这由编译器通道 `SpillSequenceSpiller` 处理，以确保它只出现在栈为空的地方。

需要断点支持的表达式的降级是在三个新序列点绑定语句节点的帮助下完成的，这些节点放置在 `BoundSpillSequence` 中：

> ```xml
>   <!--
>     这用于保存调试器在代码中此位置的"当前序列点"概念，以便之后可以通过
>     BoundRestorePreviousSequencePoint 节点恢复。当此语句出现时，
>     保存前一个非隐藏序列点并将其与给定 Identifier 关联。
>     -->
>   <Node Name="BoundSavePreviousSequencePoint" Base="BoundStatement">
>     <Field Name="Identifier" Type="object"/>
>   </Node>
> 
>   <!--
>     这用于将调试器的"当前语句"概念恢复到某个以前的位置，而不引入调试器停止的位置。
>     标识符必须之前已在 BoundSavePreviousSequencePoint 语句中给定。用于实现
>     表达式内的断点（例如 switch 表达式）。
>     -->
>   <Node Name="BoundRestorePreviousSequencePoint" Base="BoundStatement">
>     <Field Name="Identifier" Type="object"/>
>   </Node>
> 
>   <!--
>     这用于设置调试器的"当前语句"概念，而不会导致调试器在此处单步时停止。
>     -->
>   <Node Name="BoundStepThroughSequencePoint" Base="BoundStatement">
>     <Field Name="Span" Type="TextSpan"/>
>   </Node>
> ```

`BoundSavePreviousSequencePoint` 用于在表达式开始时保存"当前语句"信息，以便在表达式之后恢复。这是必要的，以便表达式之后出现的代码在调试器中不会显示为正在执行被插桩的表达式。

`BoundRestorePreviousSequencePoint` 和 `BoundStepThroughSequencePoint` 都旨在更改调试器对当前语句的概念，而不触发单步时调试器停止的位置。这通过以下序列来实现：

> ```none
>     ldc.i4 1
>     brtrue.s L
>     // 序列点
>     nop
>   L:
>     // 隐藏序列点
> ```

这可以在测试 `SwitchExpressionSequencePoints` 中看到。

此指令序列的目的是使存在一个具有序列点的不可达 IL 操作码（`nop`），后跟一个隐藏序列点，以防止调试器停在以下位置。但是，一旦执行以下代码，调试器对程序"当前源位置"的概念将显示为序列点中提到的位置。

`BoundStepThroughSequencePoint` 也修改调试器对"封闭语句"的视图，但不创建可以设置断点的位置。在评估 *switch 表达式* 的状态机时，这用于使"当前语句"显示为 *switch 表达式*。
