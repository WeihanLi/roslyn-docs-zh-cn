# 运行时异步设计

另见此特性的 ECMA-335 规范变更：https://github.com/dotnet/runtime/blob/main/docs/design/specs/runtime-async.md。https://github.com/dotnet/runtime/issues/109632 跟踪运行时中该特性的未解决问题。

本文档介绍 Roslyn 如何与运行时异步特性协作生成 IL 的总体设计。我们尽量避免在用户层面暴露此特性；初始绑定几乎完全不受运行时异步影响。公开的符号不直接提供关于是否使用运行时异步编译的信息，实际上编译器也无法知道引用程序集中的方法是否使用运行时异步编译。

## 支持的运行时 API

我们使用以下辅助 API 向运行时指示挂起点，以及运行时异步调用语法：

```cs
namespace System.Runtime.CompilerServices;

namespace System.Runtime.CompilerServices;

[System.Diagnostics.CodeAnalysis.ExperimentalAttribute("SYSLIB5007", UrlFormat = "https://aka.ms/dotnet-warnings/{0}")]
public static partial class AsyncHelpers
{
    public static void UnsafeAwaitAwaiter<TAwaiter>(TAwaiter awaiter) where TAwaiter : ICriticalNotifyCompletion { }
    public static void AwaitAwaiter<TAwaiter>(TAwaiter awaiter) where TAwaiter : INotifyCompletion { }

    // 这些方法用于直接 await 方法调用
    public static void Await(System.Threading.Tasks.Task task) { }
    public static T Await<T>(System.Threading.Tasks.Task<T> task) { }
    public static void Await(System.Threading.Tasks.ValueTask task) { }
    public static T Await<T>(System.Threading.Tasks.ValueTask<T> task) { }
    public static void Await(System.Runtime.CompilerServices.ConfiguredTaskAwaitable configuredAwaitable) { }
    public static T Await<T>(System.Runtime.CompilerServices.ConfiguredTaskAwaitable<T> configuredAwaitable) { }
    public static void Await(System.Runtime.CompilerServices.ConfiguredValueTaskAwaitable configuredAwaitable) { }
    public static T Await<T>(System.Runtime.CompilerServices.ConfiguredValueTaskAwaitable<T> configuredAwaitable) { }
}
```

这些 API 的存在也驱动了特性是否可以使用。这些 API 必须在定义 `object` 的同一程序集中定义，且该程序集不能引用任何其他程序集。就 CoreFX 而言，这意味着必须在 `System.Runtime` 引用程序集中定义。

我们假设在 `AsyncHelpers` 定义时存在以下 `MethodImplOptions` 位，用于告知 JIT 为该方法生成异步状态机。此位不允许在任何方法上手动使用；它由编译器添加到 `async` 方法。

TODO：我们可能希望阻止直接调用具有非 `Task`/`ValueTask` 返回类型的 `MethodImplOptions.Async` 方法。

```cs
namespace System.Runtime.CompilerServices;

public enum MethodImplOptions
{
    Async = 0x2000
}
```

出于实验目的，我们识别一个可用于强制编译器生成运行时异步代码或强制生成完整状态机的属性。此属性未在 BCL 中定义，作为实验的逃生舱。当特性在稳定版中发布时，可能会被移除。

```cs
namespace System.Runtime.CompilerServices;

[AttributeUsage(AttributeTargets.Method)]
public class RuntimeAsyncMethodGenerationAttribute(bool runtimeAsync) : Attribute();
```

## 转换策略

如前所述，我们尽量避免将此暴露给初始绑定。唯一的主要例外是对 `MethodImplOption.Async` 的处理；我们不允许将其应用于用户代码，如果用户手动尝试，将发出错误。

编译器生成的异步状态机和运行时生成的异步共享一些相同的构建块。两者都需要将 `catch` 和 `finally` 块内的 `await` 重写为挂起异常，在 `catch`/`finally` 区域外执行 `await`，然后根据需要恢复异常。

TODO：回顾 `IAsyncEnumerable` 并确认初始重写为基于 `Task` 的方法生成的代码可以用运行时异步实现，而不是完整的编译器状态机。

TODO：与调试器团队澄清在调试/ENC 场景中需要在哪里插入 NOP。对于我们向 `AsyncHelpers` 辅助方法发出调用的场景，可能需要插入 AwaitYieldPoint 和 AwaitResumePoints，但对于运行时异步形式的调用，是否可以避免？

### 示例转换

以下是针对特定示例生成 IL 的一些示例。

TODO：包含调试版本

#### 一般签名转换

总体而言，C# 中声明的 async 方法将按如下方式转换：

```cs
async Task M()
{
    // ...
}
```

```cs
[MethodImpl(MethodImplOptions.Async)]
Task M()
{
  // ... 见下面各种 await 类型的降级策略 ...
}
```

返回 `Task<T>`、`ValueTask` 和 `ValueTask<T>` 的方法也适用相同规则。任何返回不同 Task 类似类型的方法都不会转换为运行时异步形式，而使用 C# 生成的状态机。

方法体内的 `await` 将被转换为运行时异步调用格式（详见运行时规范），或者我们使用 `AsyncHelpers` 方法之一来执行 `await`。具体场景详见下文。

TODO：异步迭代器（返回 `IAsyncEnumerable<T>`）

#### `AsyncHelpers.Await` 场景

对于类型为 `E` 的 `await expr`，编译器将尝试在 `System.Runtime.CompilerServices.AsyncHelpers` 中匹配辅助方法，使用以下算法：

1. 如果 `E` 的泛型元数大于 1，则未找到匹配，转到[await 任何其他类型]。
2. 从 corelib（定义 `System.Object` 且没有引用的库）获取 `System.Runtime.CompilerServices.AsyncHelpers`。
3. 将所有名为 `Await` 的方法放入名为 `M` 的组。
4. 对于 `M` 中的每个 `Mi`：
   1. 如果 `Mi` 的泛型元数与 `E` 不匹配，则将其移除。
   2. 如果 `Mi` 接受超过 1 个参数（命名为 `P`），则将其移除。
   3. 如果 `Mi` 的泛型元数为 0，以下所有条件必须为真，否则移除 `Mi`：
      1. 返回类型为 `System.Void`
      2. 从 `E` 到 `P` 的类型存在标识或隐式引用转换。
   4. 否则，如果 `Mi` 的泛型元数为 1，类型参数为 `Tm`，以下所有条件必须为真，否则移除 `Mi`：
      1. 返回类型为 `Tm`
      2. `E` 的泛型参数为 `Te`
      3. `Ti` 满足 `Tm` 上的所有约束
      4. `Mie` 是将 `Te` 替换 `Tm` 后的 `Mi`，`Pe` 是 `Mie` 的结果参数
      5. 从 `E` 到 `Pe` 的类型存在标识或隐式引用转换
5. 如果只剩一个 `Mi`，则使用该方法进行以下重写。否则，转到[await 任何其他类型]。

我们通常将 `await expr` 重写为 `System.Runtime.CompilerServices.AsyncHelpers.Await(expr)`。下面涵盖了若干不同的示例场景，主要有趣的偏差是 `struct` 右值需要跨 `await` 提升，以及异常处理重写。

这些规则旨在涵盖以下类型：`Task` 或其任何子类、`Task<T>` 或其任何子类、`ValueTask`、`ValueTask<T>`、`ConfiguredTaskAwaitable`、`ConfiguredTaskAwaitable<T>`、`ConfiguredValueTaskAwaitable`、`ConfiguredValueTaskAwaitable<T>`，以及运行时希望内联化的任何未来 Task 类似类型。

##### await 返回 `Task` 的方法

```cs
class C
{
    static Task M();
}

await C.M();
```

转换后的 C#：

```cs
System.Runtime.CompilerServices.AsyncHelpers.Await(C.M());
```

```il
call [System.Runtime]System.Threading.Tasks.Task C::M()
call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
```

---

```cs
var c = new C();
await c.M();

class C
{
    Task M();
}
```

转换后的 C#：

```cs
var c = new C();
System.Runtime.CompilerServices.AsyncHelpers.Await(c.M());
```

```il
newobj instance void C::.ctor()
callvirt instance class [System.Runtime]System.Threading.Tasks.Task C::M()
call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
```

<details>
<summary>await 简单 `expr` 场景的更多扩展示例</summary>

##### await 返回具体 `T` 的 `Task<T>` 方法

```cs
int i = await C.M();

class C
{
    static Task<int> M();
}
```

转换后的 C#：

```cs
int i = System.Runtime.CompilerServices.AsyncHelpers.Await<int>(C.M());
```

```il
call class [System.Runtime]System.Threading.Tasks.Task`1<int32> C::M()
call int32 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<int32>)
stloc.0
```

---

```cs
var c = new C();
int i = await c.M();

class C
{
    Task<int> M();
}
```

转换后的 C#：

```cs
var c = new C();
int i = System.Runtime.CompilerServices.AsyncHelpers.Await<int>(c.M());
```

```il
newobj instance void C::.ctor()
callvirt instance class [System.Runtime]System.Threading.Tasks.Task`1<int32> C::M()
call int32 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<int32>)
stloc.0
```

##### await `Task` 类型的局部变量

```cs
var local = M();
await local;

class C
{
    static Task M();
}
```

转换后的 C#：

```cs
var local = C.M();
System.Runtime.CompilerServices.AsyncHelpers.Await(local);
```

```il
{
    .locals init (
        [0] valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter awaiter
    )

    IL_0000: call class [System.Runtime]System.Threading.Tasks.Task C::M()
    IL_0005: stloc.0
    IL_0006: ldloc.0
    IL_0007: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_000c: ret
}
```

##### await 具体类型 `Task<T>` 的局部变量

```cs
var local = M();
var i = await local;

class C
{
    static Task<int> M();
}
```

转换后的 C#：

```cs
var local = C.M();
var i = System.Runtime.CompilerServices.AsyncHelpers.Await<int>(local);
```

```il
{
    .locals init (
        [0] class [System.Runtime]System.Threading.Tasks.Task`1<int32> local,
        [1] int32 i
    )

    IL_0000: call class [System.Runtime]System.Threading.Tasks.Task`1<int32> C::M()
    IL_0005: stloc.0
    IL_0006: ldloc.0
    IL_0007: call !!0 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<!!0>)
    IL_000c: stloc.1
    IL_000d: ret
}
```

##### await 返回 `T` 的方法

```cs
await C.M<Task>();

class C
{
    static T M<T>();
}
```

转换后的 C#：

```cs
System.Runtime.CompilerServices.AsyncHelpers.Await(C.M<Task>());
```

```il
{
    IL_0000: call !!0 C::M<class [System.Runtime]System.Threading.Tasks.Task>()
    IL_0005: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_000a: ret
}
```

##### await 返回泛型 `T` 的 `Task<T>` 方法

```cs
int i = await C.M<int>();

class C
{
    static Task<T> M<T>();
}
```

转换后的 C#：

```cs
int i = System.Runtime.CompilerServices.AsyncHelpers.Await<int>(C.M<int>());
```

```il
{
    IL_0000: call class [System.Runtime]System.Threading.Tasks.Task`1<!!0> C::M<int32>()
    IL_0005: call !!0 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<!!0>)
    IL_000a: stloc.0
    IL_000b: ret
}
```

##### await 返回 `Task` 的委托

```cs
AsyncDelegate d = C.M;
await d();

delegate Task AsyncDelegate();

class C
{
    static Task M();
}
```

转换后的 C#：

```cs
AsyncDelegate d = C.M;
System.Runtime.CompilerServices.AsyncHelpers.Await(d());
```

```il
{
    IL_0000: ldsfld class AsyncDelegate Program/'<>O'::'<0>__M'
    IL_0005: dup
    IL_0006: brtrue.s IL_001b

    IL_0008: pop
    IL_0009: ldnull
    IL_000a: ldftn class [System.Runtime]System.Threading.Tasks.Task C::M()
    IL_0010: newobj instance void AsyncDelegate::.ctor(object, native int)
    IL_0015: dup
    IL_0016: stsfld class AsyncDelegate Program/'<>O'::'<0>__M'

    IL_001b: callvirt instance class [System.Runtime]System.Threading.Tasks.Task AsyncDelegate::Invoke()
    IL_0020: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_0025: ret
}
```

##### await 返回 `T` 的委托（`T` 为 `Task`）

```cs
Func<Task> d = C.M;
await d();

class C
{
    static Task M();
}
```

转换后的 C#：

```cs
Func<Task> d = C.M;
System.Runtime.CompilerServices.AsyncHelpers.Await(d());
```

```il
{
    IL_0000: ldsfld class [System.Runtime]System.Func`1<class [System.Runtime]System.Threading.Tasks.Task> Program/'<>O'::'<0>__M'
    IL_0005: dup
    IL_0006: brtrue.s IL_001b

    IL_0008: pop
    IL_0009: ldnull
    IL_000a: ldftn class [System.Runtime]System.Threading.Tasks.Task C::M()
    IL_0010: newobj instance void class [System.Runtime]System.Func`1<class [System.Runtime]System.Threading.Tasks.Task>::.ctor(object, native int)
    IL_0015: dup
    IL_0016: stsfld class [System.Runtime]System.Func`1<class [System.Runtime]System.Threading.Tasks.Task> Program/'<>O'::'<0>__M'

    IL_001b: callvirt instance !0 class [System.Runtime]System.Func`1<class [System.Runtime]System.Threading.Tasks.Task>::Invoke()
    IL_0020: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_0025: ret
}
```

</details>

##### 在 `catch` 块中 await

```cs
try
{
    throw new Exception();
}
catch (Exception ex)
{
    await C.M();
    throw;
}

class C
{
    static Task M();
}
```

转换后的 C#：

```cs
int pendingCatch = 0;
Exception pendingException;
try
{
    throw new Exception();
}
catch (Exception e)
{
    pendingCatch = 1;
    pendingException = e;
}

if (pendingCatch == 1)
{
    System.Runtime.CompilerServices.AsyncHelpers.Await(C.M());
    throw pendingException;
}
```

```il
{
    .locals init (
        [0] int32 pendingCatch,
        [1] class [System.Runtime]System.Exception pendingException
    )

    IL_0000: ldc.i4.0
    IL_0001: stloc.0
    .try
    {
        IL_0002: newobj instance void [System.Runtime]System.Exception::.ctor()
        IL_0007: throw
    } // end .try
    catch [System.Runtime]System.Exception
    {
        IL_0008: ldc.i4.1
        IL_0009: stloc.0
        IL_000a: stloc.1
        IL_000b: leave.s IL_000d
    } // end handler

    IL_000d: ldloc.0
    IL_000e: ldc.i4.1
    IL_000f: bne.un.s IL_001d

    IL_0011: call class [System.Runtime]System.Threading.Tasks.Task C::M()
    IL_0016: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_001b: ldloc.1
    IL_001c: throw

    IL_001d: ret
}
```

##### 在 `finally` 块中 await

```cs
try
{
    throw new Exception();
}
finally
{
    await C.M();
}

class C
{
    static Task M();
}
```

转换后的 C#：

```cs
Exception pendingException;
try
{
    throw new Exception();
}
catch (Exception e)
{
    pendingException = e;
}

System.Runtime.CompilerServices.AsyncHelpers.Await(C.M());

if (pendingException != null)
{
    throw pendingException;
}
```

```il
{
    .locals init (
        [0] class [System.Runtime]System.Exception pendingException
    )

    .try
    {
        IL_0000: newobj instance void [System.Runtime]System.Exception::.ctor()
        IL_0005: throw
    } // end .try
    catch [System.Runtime]System.Exception
    {
        IL_0006: stloc.0
        IL_0007: leave.s IL_0009
    } // end handler

    IL_0009: call class [System.Runtime]System.Threading.Tasks.Task C::M()
    IL_000e: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_0013: ldloc.0
    IL_0014: brfalse.s IL_0018

    IL_0016: ldloc.0
    IL_0017: throw

    IL_0018: ret
}
```

##### 保留复合赋值

```cs
int[] a = new int[] { };
a[C.M2()] += await C.M1();

class C
{
    public static Task<int> M1();
    public static int M2();
}
```

转换后的 C#：

```cs
int[] a = new int[] { };
int _tmp1 = C.M2();
int _tmp2 = a[_tmp1];
int _tmp3 = System.Runtime.CompilerServices.AsyncHelpers.Await(C.M1());
a[_tmp1] = _tmp2 + _tmp3;
```

```il
{
    .locals init (
        [0] int32 _tmp1,
        [1] int32 _tmp2,
        [2] int32 _tmp3
    )

    IL_0000: ldc.i4.0
    IL_0001: newarr [System.Runtime]System.Int32
    IL_0006: call int32 C::M2()
    IL_000b: stloc.0
    IL_000c: dup
    IL_000d: ldloc.0
    IL_000e: ldelem.i4
    IL_000f: stloc.1
    IL_0010: call class [System.Runtime]System.Threading.Tasks.Task`1<int32> C::M1()
    IL_0015: call !!0 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<!!0>)
    IL_001a: stloc.2
    IL_001b: ldloc.0
    IL_001c: ldloc.1
    IL_001d: ldloc.2
    IL_001e: add
    IL_001f: stelem.i4
    IL_0020: ret
}
```

#### await 任何其他类型

对于非 `Task`、`Task<T>`、`ValueTask` 和 `ValueTask<T>` 的类型，我们改用 `System.Runtime.CompilerServices.AsyncHelpers.AwaitAwaiterFromRuntimeAsync` 或 `System.Runtime.CompilerServices.AsyncHelpers.UnsafeAwaitAwaiterFromRuntimeAsync`。

##### 实现 ICriticalNotifyCompletion 的类型

当静态已知表达式实现了 `ICriticalNotifyCompletion` 时，始终优先使用 `ICriticalNotifyCompletion` 降级，而不是 `INotifyCompletion` 降级。

```cs
var c = new C();
await c;

class C
{
    public class Awaiter : ICriticalNotifyCompletion
    {
        public void OnCompleted(Action continuation) { }
        public void UnsafeOnCompleted(Action continuation) { }
        public bool IsCompleted => true;
        public void GetResult() { }
    }

    public Awaiter GetAwaiter() => new Awaiter();
}
```

转换后的 C#：

```cs
var c = new C();
_ = {
    var awaiter = c.GetAwaiter();
    if (!awaiter.IsCompleted)
    {
        System.Runtime.CompilerServices.AsyncHelpers.UnsafeAwaitAwaiterFromRuntimeAsync<C.Awaiter>(awaiter);
    }
    awaiter.GetResult()
};
```

```il
{
    .locals init (
        [0] class C/Awaiter awaiter
    )

    IL_0000: newobj instance void C::.ctor()
    IL_0005: callvirt instance class C/Awaiter C::GetAwaiter()
    IL_000a: stloc.0
    IL_000b: ldloc.0
    IL_000c: callvirt instance bool C/Awaiter::get_IsCompleted()
    IL_0011: brtrue.s IL_0019

    IL_0013: ldloc.0
    IL_0014: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::UnsafeAwaitAwaiterFromRuntimeAsync<class C/Awaiter>(!!0)

    IL_0019: ldloc.0
    IL_001a: callvirt instance void C/Awaiter::GetResult()
    IL_001f: ret
}
```

##### 实现 INotifyCompletion 的类型

```cs
var c = new C();
await c;

class C
{
    public class Awaiter : INotifyCompletion
    {
        public void OnCompleted(Action continuation) { }
        public bool IsCompleted => true;
        public void GetResult() { }
    }

    public Awaiter GetAwaiter() => new Awaiter();
}
```

转换后的 C#：

```cs
var c = new C();
_ = {
    var awaiter = c.GetAwaiter();
    if (!awaiter.IsCompleted)
    {
        System.Runtime.CompilerServices.AsyncHelpers.AwaitAwaiterFromRuntimeAsync<C.Awaiter>(awaiter);
    }
    awaiter.GetResult()
};
```

```il
{
    .locals init (
        [0] class C/Awaiter awaiter
    )

    IL_0000: newobj instance void C::.ctor()
    IL_0005: callvirt instance class C/Awaiter C::GetAwaiter()
    IL_000a: stloc.0
    IL_000b: ldloc.0
    IL_000c: callvirt instance bool C/Awaiter::get_IsCompleted()
    IL_0011: brtrue.s IL_0019

    IL_0013: ldloc.0
    IL_0014: call void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::AwaitAwaiterFromRuntimeAsync<class C/Awaiter>(!!0)

    IL_0019: ldloc.0
    IL_001a: callvirt instance void C/Awaiter::GetResult()
    IL_001f: ret
}
```
