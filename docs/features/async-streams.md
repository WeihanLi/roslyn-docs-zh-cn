async-streams（C# 8.0）
----------------------

异步流是可枚举对象的异步变体，其中获取下一个元素可能涉及异步操作。它们是实现了 `IAsyncEnumerable<T>` 的类型。

```csharp
// 这些接口将作为 .NET Core 3 的一部分发布
namespace System.Collections.Generic
{
    public interface IAsyncEnumerable<out T>
    {
        IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken token = default);
    }

    public interface IAsyncEnumerator<out T> : System.IAsyncDisposable
    {
        System.Threading.Tasks.ValueTask<bool> MoveNextAsync();
        T Current { get; }
    }
}
namespace System
{
    public interface IAsyncDisposable
    {
        System.Threading.Tasks.ValueTask DisposeAsync();
    }
}
```

当你拥有一个异步流时，可以使用异步 `foreach` 语句枚举其元素：`await foreach (var item in asyncStream) { ... }`。
`await foreach` 语句与 `foreach` 语句类似，但它使用 `IAsyncEnumerable` 而非 `IEnumerable`，每次迭代都会求值 `await MoveNextAsync()`，且枚举器的释放是异步的。

类似地，如果你有一个异步可释放对象，可以使用异步 `using` 语句来使用并释放它：`await using (var resource = asyncDisposable) { ... }`
`await using` 语句与 `using` 语句类似，但它使用 `IAsyncDisposable` 而非 `IDisposable`，并使用 `await DisposeAsync()` 而非 `Dispose()`。

用户可以手动实现这些接口，也可以利用编译器从用户定义的方法（称为"async-iterator 方法"）生成状态机。
async-iterator 方法是满足以下条件的方法：
1. 声明为 `async`，
2. 返回 `IAsyncEnumerable<T>` 或 `IAsyncEnumerator<T>` 类型，
3. 同时使用 `await` 语法（`await` 表达式、`await foreach` 或 `await using` 语句）和 `yield` 语句（`yield return`、`yield break`）。

例如：
```csharp
async IAsyncEnumerable<int> GetValuesFromServer()
{
    while (true)
    {
        IEnumerable<int> batch = await GetNextBatch();
        if (batch == null) yield break;

        foreach (int item in batch)
        {
            yield return item;
        }
    }
}
```

与迭代器方法一样，在 async-iterator 方法中，`yield` 语句出现的位置有若干限制：
- `yield` 语句（任意形式）出现在 `try` 语句的 `finally` 子句中是编译时错误。
- `yield return` 语句出现在包含任何 `catch` 子句的 `try` 语句中的任何位置都是编译时错误。

### `await using` 语句的详细设计

异步 `using` 的降级处理方式与常规 `using` 相同，只是将 `Dispose()` 替换为 `await DisposeAsync()`。

注意，`DisposeAsync` 的基于模式的查找绑定到无需参数即可调用的实例方法。
扩展方法不参与其中。`DisposeAsync` 的结果必须是可等待的。

### `await foreach` 语句的详细设计

`await foreach` 的降级处理方式与常规 `foreach` 相同，但有以下区别：
- `GetEnumerator()` 替换为 `await GetAsyncEnumerator()`
- `MoveNext()` 替换为 `await MoveNextAsync()`
- `Dispose()` 替换为 `await DisposeAsync()`

注意，`GetAsyncEnumerator`、`MoveNextAsync` 和 `DisposeAsync` 的基于模式的查找绑定到无需参数即可调用的实例方法。
扩展方法不参与其中。`MoveNextAsync` 和 `DisposeAsync` 的结果必须是可等待的。
`await foreach` 的释放不包含回退到对接口实现的运行时检查。

对 dynamic 类型的集合不允许使用异步 foreach 循环，
因为不存在非泛型 `IEnumerable` 接口的异步等价物。

但包装类型可以传递非默认值（参见 `.WithCancellation(CancellationToken)` 扩展方法），
从而允许异步流的使用方控制取消。
异步流的生产方可以通过在自定义类型中编写
`IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken)` async-iterator 方法来利用取消令牌。

```csharp
E e = ((C)(x)).GetAsyncEnumerator(default);
try
{
    while (await e.MoveNextAsync())
    {
        V v = (V)(T)e.Current;  -OR-  (D1 d1, ...) = (V)(T)e.Current;
        // body
    }
}
finally
{
    await e.DisposeAsync();
}
```

### async-iterator 方法的详细设计

async-iterator 方法被替换为一个启动方法，该方法初始化状态机，但不会启动状态机的运行（与常规 async 方法的启动方法不同）。
启动方法标有 `AsyncIteratorStateMachineAttribute`。

可枚举 async-iterator 方法的状态机主要实现 `IAsyncEnumerable<T>` 和 `IAsyncEnumerator<T>`。
对于枚举器 async-iterator，它仅实现 `IAsyncEnumerator<T>`。
它类似于为 async 方法生成的状态机。
它包含 builder 和 awaiter 字段，用于在后台运行状态机（当 async-iterator 中遇到 `await` 时）。
它还捕获参数值（如有）或 `this`（如需要）。

但它包含额外状态：
- 值或结束的 promise，
- 当前产生值，类型为 `T`，
- 捕获创建线程 ID 的 `int`，
- 表示"释放模式"的 `bool` 标志，
- 用于合并令牌的 `CancellationTokenSource`（在可枚举类型中）。

状态机的核心方法是 `MoveNext()`。它由 `MoveNextAsync()` 调用，或作为从方法中 `await` 发起的后台延续调用。

值或结束的 promise 从 `MoveNextAsync` 返回。它可以通过以下方式完成：
- `true`（当状态机后台执行后值变为可用时），
- `false`（如果到达末尾），
- 异常。
promise 实现为状态机上的 `ManualResetValueTaskSourceCore<bool>`（这是一种可重用且无分配的方式来生成和完成 `ValueTask<bool>` 或 `ValueTask` 实例）
及其周围接口：`IValueTaskSource<bool>` 和 `IValueTaskSource`。
有关这些类型的更多详情，请参见 https://blogs.msdn.microsoft.com/dotnet/2018/11/07/understanding-the-whys-whats-and-whens-of-valuetask/

与常规 async 方法的状态机相比，async-iterator 方法的 `MoveNext()` 增加了以下逻辑：
- 支持处理 `yield return` 语句，保存当前值并以结果 `true` 完成 promise，
- 支持处理 `yield break` 语句，设置释放模式并跳转到封闭的 `finally` 或退出，
- 调度执行到 `finally` 块（释放时），
- 退出方法，释放 `CancellationTokenSource`（如有）并以结果 `false` 完成 promise，
- 捕获异常，释放 `CancellationTokenSource`（如有）并将异常设置到 promise 中。
（`await` 的处理保持不变）

这反映在实现中，它扩展了 async 方法的降级机制：
1. 处理 `yield return` 和 `yield break` 语句（参见 `AsyncIteratorMethodToStateMachineRewriter` 中的 `VisitYieldReturnStatement` 和 `VisitYieldBreakStatement` 方法），
2. 处理 `try` 语句（参见 `AsyncIteratorMethodToStateMachineRewriter` 中的 `VisitTryStatement` 和 `VisitExtractedFinallyBlock` 方法）
3. 为 promise 本身生成额外的状态和逻辑（参见 `AsyncIteratorRewriter`，它生成各种其他成员：`MoveNextAsync`、`Current`、`DisposeAsync`，
以及支持可重置 `ValueTask` 行为的一些成员，即 `GetResult`、`SetStatus`、`OnCompleted`）。

```csharp
ValueTask<bool> MoveNextAsync()
{
    if (state == StateMachineStates.FinishedStateMachine)
    {
        return default(ValueTask<bool>);
    }
    valueOrEndPromise.Reset();
    var inst = this;
    builder.Start(ref inst);
    var version = valueOrEndPromise.Version;
    if (valueOrEndPromise.GetStatus(version) == ValueTaskSourceStatus.Succeeded)
    {
        return new ValueTask<bool>(valueOrEndPromise.GetResult(version));
    }
    return new ValueTask<bool>(this, version); // note this leverages the state machine's implementation of IValueTaskSource<bool>
}
```

```csharp
T Current => current;
```

async-iterator 方法的启动方法和状态机初始化与常规迭代器方法相同。
特别地，合成的 `GetAsyncEnumerator()` 方法类似于 `GetEnumerator()`，但将初始状态设置为 StateMachineStates.NotStartedStateMachine (-1)：
```csharp
IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken token)
{
    {StateMachineType} result;
    if (initialThreadId == /*managedThreadId*/ && state == StateMachineStates.FinishedStateMachine)
    {
        state = InitialState; // -3
        builder = AsyncIteratorMethodBuilder.Create();
        disposeMode = false;
        result = this;
    }
    else
    {
        result = new {StateMachineType}(InitialState);
    }
    /* copy each parameter proxy, or in the case of the parameter marked with [EnumeratorCancellation] combine it with `GetAsyncEnumerator`'s `token` parameter */
}
```

对于带有 `[EnumeratorCancellation]` 的参数，`GetAsyncEnumerator` 通过合并两个可用令牌来初始化它：
```csharp
if (this.parameterProxy.Equals(default))
{
    result.parameter = token;
}
else if (token.Equals(this.parameterProxy) || token.Equals(default))
{
    result.parameter = this.parameterProxy;
}
else
{
    result.combinedTokens = CancellationTokenSource.CreateLinkedTokenSource(this.parameterProxy, token);
    result.parameter = combinedTokens.Token;
}
```
有关线程 ID 检查的讨论，请参见 https://github.com/dotnet/corefx/issues/3481

同样，启动方法与常规迭代器方法非常相似：
```csharp
{
    {StateMachineType} result = new {StateMachineType}(StateMachineStates.FinishedStateMachine); // -2
    /* save parameters into parameter proxies */
    return result;
}
```

#### 释放

迭代器和 async-iterator 方法需要释放，因为它们的执行步骤由调用方控制，调用方可能在获取所有元素之前选择释放枚举器。
例如，`foreach (...) { if (...) break; }`。
相比之下，async 方法会自主运行直到完成。从调用方角度来看，它们永远不会在执行中途挂起，因此不需要释放。

总结来说，async-iterator 的释放基于四个设计要素：
- `yield return`（在释放模式下恢复时跳转到 finally）
- `yield break`（进入释放模式并跳转到封闭的 finally）
- `finally`（在一个 `finally` 之后跳转到下一个封闭的 finally）
- `DisposeAsync`（进入释放模式并恢复执行）

async-iterator 方法的调用方只应在方法完成或被 `yield return` 挂起时调用 `DisposeAsync()`。
`DisposeAsync` 在状态机上设置一个标志（"释放模式"），并（如果方法未完成）从当前状态恢复执行。
状态机可以从给定状态恢复执行（即使是位于 `try` 内的状态）。
当在释放模式下恢复执行时，它直接跳转到封闭的 `finally`。
`finally` 块可能涉及暂停和恢复，但仅针对 `await` 表达式。由于对 `yield return` 施加的限制（如上所述），释放模式永远不会遇到 `yield return`。
一旦 `finally` 块完成，释放模式下的执行跳转到下一个封闭的 `finally`，或在到达顶层后跳转到方法末尾。

到达 `yield break` 也会设置释放模式标志并跳转到封闭的 `finally`（或方法末尾）。
当我们将控制权返回给调用方时（通过到达方法末尾以 `false` 完成 promise），所有释放已完成，
状态机处于完成状态。因此 `DisposeAsync()` 没有剩余工作要做。

从给定 `finally` 块的角度来看，该块中的代码可以通过以下方式执行：
- 正常执行（即 `try` 块中的代码执行完毕后），
- `try` 块内引发异常（这将执行必要的 `finally` 块并使方法以完成状态终止），
- 调用 `DisposeAsync()`（在释放模式下恢复执行并跳转到封闭的 finally），
- 紧随 `yield break`（进入释放模式并跳转到封闭的 finally），
- 在释放模式下，紧随嵌套的 `finally`。

`yield return` 降级为：
```csharp
_current = expression;
_state = <next_state>;
goto <exprReturnTruelabel>; // which does _valueOrEndPromise.SetResult(true); return;

// resuming from state=<next_state> will dispatch execution to this label
<next_state_label>: ;
this.state = cachedState = NotStartedStateMachine;
if (disposeMode) /* jump to enclosing finally or exit */
```

`yield break` 降级为：
```csharp
disposeMode = true;
/* jump to enclosing finally or exit */
```

```csharp
ValueTask IAsyncDisposable.DisposeAsync()
{
    if (state >= StateMachineStates.NotStartedStateMachine /* -1 */)
    {
        throw new NotSupportedException();
    }
    if (state == StateMachineStates.FinishedStateMachine /* -2 */)
    {
        return default;
    }
    disposeMode = true;
    _valueOrEndPromise.Reset();
    var inst = this;
    _builder.Start(ref inst);
    return new ValueTask(this, _valueOrEndPromise.Version);  // note this leverages the state machine's implementation of IValueTaskSource
}
```

##### 常规 finally 与提取的 finally

当 `finally` 子句不包含 `await` 表达式时，`try/finally` 降级为：
```csharp
try
{
    ...
    finallyEntryLabel:
}
finally
{
    ...
}
if (disposeMode) /* jump to enclosing finally or exit */
```

当 `finally` 包含 `await` 表达式时，它在 async 重写之前被提取（由 AsyncExceptionHandlerRewriter）。在这些情况下，我们得到：
```csharp
try
{
    ...
    goto finallyEntryLabel;
}
catch (Exception e)
{
    ... save exception ...
}
finallyEntryLabel:
{
    ... original code from finally and additional handling for exception ...
}
```

在两种情况下，我们都会在 `finally` 逻辑块之后添加 `if (disposeMode) /* jump to enclosing finally or exit */`。

#### 状态值与转换

可枚举对象以状态 -2 开始。
调用 GetAsyncEnumerator 将状态设置为 -3，或返回一个新的枚举器（也具有状态 -3）。

从那里，MoveNext 将：
- 到达方法末尾（-2，完成并释放）
- 到达 `yield break`（状态不变，释放模式 = true）
- 到达 `yield return`（-N，从 -4 递减）
- 到达 `await`（N，从 0 递增）

从挂起状态 N 或 -N，MoveNext 将恢复执行（-1）。
但如果挂起是 `yield return`（-N），你也可以调用 DisposeAsync，它在释放模式下恢复执行（-1）。

在释放模式下，MoveNext 继续挂起（N）和恢复（-1），直到到达方法末尾（-2）。

从状态 -1 或 N 调用 `DisposeAsync` 的结果未指定。编译器对这些情况生成 `throw new NotSupportException()`。

```
        DisposeAsync                              await
 +------------------------+             +------------------------> N
 |                        |             |                          |
 v   GetAsyncEnumerator   |             |        resuming          |
-2 --------------------> -3 --------> -1 <-------------------------+    Dispose mode = false
 ^                                   |  |                          |
 |         done and disposed         |  |      yield return        |
 +-----------------------------------+  +-----------------------> -N
 |        or exception thrown        |                             |
 |                                   |                             |
 |                             yield |                             |
 |                             break |           DisposeAsync      |
 |                                   |  +--------------------------+
 |                                   |  |
 |                                   |  |
 |         done and disposed         v  v    suspension (await)
 +----------------------------------- -1 ------------------------> N
                                        ^                          |    Dispose mode = true
                                        |         resuming         |
                                        +--------------------------+
```
