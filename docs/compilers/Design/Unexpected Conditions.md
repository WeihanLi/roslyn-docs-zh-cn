以下是编写可能遇到"不可能发生"和意外程序条件的代码的[指南](https://www.youtube.com/watch?v=6GMkuPiIZ2k)。与其他*编码指南*一样，即使我们的观点可能有所不同，在整个代码库中保持一定程度的统一也是有价值的。当你有充分理由偏离这些指南时，请添加注释，向后来者解释偏离的原因。

- 使用 `Debug.Assert(Condition)` 来记录代码中的不变量。尽管这些断言在非 Debug 构建中会消失，但作为一个开源项目，我们对客户是否在启用 Debug 的情况下运行我们的代码几乎没有控制权。因此，此类断言不应消耗过多的执行时间。我们将来可能会考虑在生产构建中运行此类断言。如果我们发现它们过于昂贵，可能会创建工作项来提高其性能。
- 如果你编写了一个旨在穷举所有可能性的 switch 语句，请添加一个带有 `throw ExceptionUtilities.UnexpectedValue(switchExpression);` 的 default 分支。
- 类似地，如果你有一系列旨在穷举所有可能性的 `if-then-else if` 语句，请在最后添加一个"不可能到达"的 else 分支，内容为 `throw ExceptionUtilities.Unreachable();`；或者，如果能方便地为诊断信息提供有趣且易于获取的数据，则使用 `ExceptionUtilities.UnexpectedValue`。对于代码到达"不可能"位置的其他情况，也要这样处理。
- 公共 API 中前置条件的验证应使用普通代码完成。如果存在适用于该情况的诊断（例如语法错误），则报告诊断；否则抛出适当的异常，例如 `ArgumentNullException` 或 `ArgumentException`。
- 如果遇到其他会致命的奇怪错误情况，请抛出 `InvalidOperationException`，并在消息中包含有趣且直接可获取的数据。这类情况应该很少见，并应附有注释，解释为什么使用 `Assert` 还不够充分。

遵循这些指南后，你应该找不到任何理由编写 `Debug.Assert(false)`。如果你在代码中发现了这种情况，请考虑上述哪条原则提示了一种不同的处理方式。
