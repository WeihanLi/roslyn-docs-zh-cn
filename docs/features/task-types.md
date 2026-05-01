C# 中的异步任务类型
======================
扩展 `async` 以支持匹配特定模式的_任务类型_，不仅限于已知类型 `System.Threading.Tasks.Task` 和 `System.Threading.Tasks.Task<T>`。

## 任务类型
_任务类型_是带有关联_构建器类型_的 `class` 或 `struct`，通过 `System.Runtime.CompilerServices.AsyncMethodBuilderAttribute` 标识。
_任务类型_可以是非泛型（用于不返回值的 async 方法），也可以是泛型（用于返回值的方法）。

为支持 `await`，_任务类型_必须有一个对应的、可访问的 `GetAwaiter()` 方法，
该方法返回_等待器类型_的实例（参见 _C# 7.7.7.1 可等待表达式_）。

```cs
[AsyncMethodBuilder(typeof(MyTaskMethodBuilder<>))]
class MyTask<T>
{
    public Awaiter<T> GetAwaiter();
}

class Awaiter<T> : INotifyCompletion
{
    public bool IsCompleted { get; }
    public T GetResult();
    public void OnCompleted(Action completion);
}
```

## 构建器类型
_构建器类型_是与特定_任务类型_对应的 `class` 或 `struct`。
_构建器类型_最多只能有 1 个类型参数，且不能嵌套在泛型类型中。
_构建器类型_具有以下 `public` 方法。
对于非泛型_构建器类型_，`SetResult()` 没有参数。

```cs
class MyTaskMethodBuilder<T>
{
    public static MyTaskMethodBuilder<T> Create();

    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine;

    public void SetStateMachine(IAsyncStateMachine stateMachine);
    public void SetException(Exception exception);
    public void SetResult(T result);

    public void AwaitOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine;
    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : ICriticalNotifyCompletion
        where TStateMachine : IAsyncStateMachine;

    public MyTask<T> Task { get; }
}
```

## 执行
编译器使用以上类型为 `async` 方法的状态机生成代码。
（生成的代码等同于为返回 `Task`、`Task<T>` 或 `void` 的 async 方法生成的代码。
区别在于，对于那些已知类型，_构建器类型_也是编译器已知的。）

`Builder.Create()` 被调用以创建_构建器类型_的实例。

如果状态机以 `struct` 实现，则会调用 `builder.SetStateMachine(stateMachine)`，传入状态机的装箱实例，构建器可缓存该实例。

`builder.Start(ref stateMachine)` 被调用，将构建器与编译器生成的状态机实例关联。
构建器必须在 `Start()` 中或 `Start()` 返回后调用 `stateMachine.MoveNext()` 以推进状态机。
`Start()` 返回后，`async` 方法调用 `builder.Task` 以获取从 async 方法返回的任务。

每次调用 `stateMachine.MoveNext()` 都会推进状态机。
如果状态机成功完成，则调用 `builder.SetResult()`，如果有方法返回值则传入该值。
如果状态机中抛出异常，则调用 `builder.SetException(exception)`。

如果状态机到达 `await expr` 表达式，则调用 `expr.GetAwaiter()`。
如果等待器实现了 `ICriticalNotifyCompletion` 且 `IsCompleted` 为 false，
则状态机调用 `builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine)`。
`AwaitUnsafeOnCompleted()` 应调用 `awaiter.OnCompleted(action)`，传入一个在等待器完成时调用 `stateMachine.MoveNext()` 的 action。
`INotifyCompletion` 和 `builder.AwaitOnCompleted()` 的行为类似。

## 重载解析
重载解析已扩展，除 `Task` 和 `Task<T>` 外，还可识别_任务类型_。

没有返回值的 `async` lambda 与非泛型_任务类型_的重载候选参数精确匹配，
返回类型为 `T` 的 `async` lambda 与泛型_任务类型_的重载候选参数精确匹配。

否则，如果 `async` lambda 与两个候选_任务类型_参数中的任何一个都不精确匹配，或两个都精确匹配，且有从一个候选类型到另一个的隐式转换，则来源候选获胜。否则递归评估 `Task1<A>` 和 `Task2<B>` 中的类型 `A` 和 `B` 以获得更好匹配。

否则，如果 `async` lambda 与两个候选_任务类型_参数都不精确匹配，但一个候选是比另一个更特化的类型，则更特化的候选获胜。
