# 增强型 using

增强型 using 由若干相关功能组成，旨在使可释放编码模式对实现者更易参与，对最终用户更易使用。

 - _[Using 声明](#Using-declarations)_ 旨在通过允许将 `using` 添加到局部变量声明，使使用可释放类型更加简便。

 - _[基于模式的异步释放](#Pattern-based-asynchronous-disposal)_ 允许类型在无需实现 `IAsyncDisposable` 接口的情况下，选择加入异步释放。

 - _[ref struct 的基于模式的释放](#Pattern-based-disposal-for-ref-structs)_ 允许 `ref struct` 在无需实现 `IDisposable` 接口的情况下，选择加入释放。

**参见**：CSharpLang 中的[对应提案](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-8.0/using.md)。

## Using 声明

Using 声明允许用户在局部变量声明语句中指定 `using` 关键字：

```csharp
using var x = ...
// other statements
```

这等价于在同一位置将局部变量声明在 `using` 语句内部：

```csharp
using (var x = ...) 
{
    // other statements
}
```

声明没有初始化表达式的 using 声明是无效的：

```csharp
using IDisposable x; //error CS0210: You must provide an initializer in a fixed or using statement declaration

```

初始化表达式必须产生被视为可释放的类型。即该表达式在直接用于 `using` 语句时也必须有效：

```csharp
using var x = <expression> 

using (<expression>) { } // expression must also be valid in a standard using statement
```

### 生命周期

局部变量的生命周期延伸到其声明所在的作用域；在变量超出作用域之前，它将被释放。

```csharp
if (...)
{
    using var x = ...;
    
    // other statements

    // Dispose x
}
```

在同一作用域内的 using 声明按与声明顺序相反的顺序释放

```csharp
{
    using var x = ...;
    using var y = ...,  z = ...;

    // Dispose z
    // Dispose y
    // Dispose x
}
```

与作为 `using` 语句一部分声明的局部变量一样，using 局部变量是 `readonly` 的，声明后不能重新赋值。

```csharp
using var x = ...;
x = ...; // error CS1656: Cannot assign to 'x' because it is a 'using variable'
```

using 局部变量可以在赋值或声明的右侧使用。这意味着可以捕获一个生命周期比 using 局部变量更长的引用。当 using 局部变量超出作用域时，该引用仍将被释放。在释放后与该引用交互是未定义行为，但在大多数情况下预期会抛出 `ObjectDisposedException`。

```csharp
IDisposable y = null;
if (...)
{
    using var x = ...;
    y = x;
    // Dispose x
}
y.Dispose(); // undefined. ObjectDisposedException in most cases
```

可以使用已有引用作为 using 局部变量声明的初始化器。如上所述，当声明的局部变量超出作用域时，该引用将被释放，但已有引用仍可通过其之前的声明使用。

```csharp
IDisposable y = ...;
if (...)
{
    using var x = y; 
    // Dispose x, which points to the same object as y
}
y.Dispose(); // undefined. ObjectDisposedException in most cases
```

### 异步释放

在标有 `async` 的方法内部，用户可以选择在 `using` 关键字之前指定额外的 `await` 关键字，以表示异步释放：

```csharp
await using var x = ...
```

等价于
```csharp
await using (var x = ...) { }
```

初始化表达式必须产生被视为异步可释放的类型。即该表达式在直接用于 `await using` 语句时也必须有效：

```csharp
await using var x = <expression> 

await using (<expression>) { } // expression must also be valid in a standard await using statement
```

在未标有 `async` 关键字的方法中，在 using 声明上指定 `await` 关键字会导致编译器错误：

```csharp
public void M()
{
    await using var x = ...; //error CS4033: The 'await' operator can only be used within an async method. Consider marking this method with the 'async' modifier and changing its return type to 'Task'.
}
```

### 控制流

通常，控制流不受 using 声明存在的影响。但是，当 `goto` 语句及其目标 `label` 位于 using 声明的两侧时，有一些限制。

当 `goto` 或目标 `label` 与声明处于相同或更低作用域时，禁止向前跳转到 using 声明之后的位置。
```csharp
    goto label1; // error CS8648: A goto cannot jump to a location after a using declaration.
    using var x = ...;
label1:
    return;
```

```csharp
    goto label1; // ok. using declaration is in a lower scope than goto and label
    {
        using var x = ...;
    }
label1:
    return;
```

```csharp
    {
        {
            goto label1; // error CS8648: A goto cannot jump to a location after a using declaration.          
        }
        using var x = ...;
    }
label1:
    return;
```

当标签与 using 声明处于同一作用域时，禁止向后跳转到 using 声明之前的位置。
```csharp
label1:
        using var x = ...; //error CS8649: A goto cannot jump to a location before a using declaration within the same block.
        goto label1;
```

明确允许向更高作用域的标签进行向后跳转。

```csharp
label1:
    {
        using var x = ...;
        goto label1; // allowed. label1 is in a higher scope than the using declaration
    }
```

跳转到更高作用域时，任何已声明的 using 变量将在 `goto` 处被释放。尚未声明的变量**不会**被释放。
```csharp
label1:
    {
        using var x = ...;
        goto label1; // dispose x
        using var y = ...; 
    }
```

当 `goto` 和目标 `label` 不位于 using 声明两侧时，始终允许跳转到同一作用域。
```csharp
    goto label1; // ok, not jumping over a using declaration
label1:
    using var x = ...;
label2:
    goto label2; // ok, not jumping over a using declaration
```


### 其他使用限制

using 声明**不能**直接出现在 case 标签内部。它可以在 case 标签内的块中使用：

```csharp
switch (...)
{
    case ...: 
        using var x = ...; // error CS8647: A using variable cannot be used directly within a switch section (consider using braces). 
        break;
    
    case ...:
        {
            using var y = ...; // ok

            // Dispose y
        }
        break;
}
```

using 声明**不能**直接作为 `out` 变量声明的一部分出现。但可以通过在 `out` 变量之后立即添加 using 声明来轻松模拟此行为：
```csharp
if(TryGetDisposable(out var x))
{
    using var y = x;

    // Dispose y, and thus x
}
```

## 基于模式的异步释放

通过基于模式的异步释放，如果一个类型满足特定的结构要求，则无需显式实现 `IAsyncDisposable` 即可在 `await using` 语句中使用。具体要求为：

    "一个可访问的、非泛型的、返回类 Task 类型的实例方法，名为 DisposeAsync，可以不带显式参数调用"

其中可访问意味着在正常可访问性规则下，从 `await using(...)` 处合法调用。

```csharp
public class C 
{
    public static async Task M()
    {     
        await using (AsyncDisposer a = new AsyncDisposer())
        { 
        }
    }
}

public class AsyncDisposer
{
    public async ValueTask DisposeAsync() => Console.WriteLine("DisposeAsync");
}
```

在类型既可以隐式转换为 `IAsyncDisposable` 又满足异步释放模式的情况下，优先选择 `IAsyncDisposable`。

### 可选参数和 params

基于模式的异步释放方法可以包含可选参数或 `params` 参数，并仍满足模式要求。一般而言，如果在 `await using` 语法处能够以正常语言规则写出有效的 `c.DisposeAsync()` 调用，则该类型被视为异步可释放。

```csharp

public class AsyncDisposer
{
    public async ValueTask DisposeAsync(int x = 0, params object[] args) => Console.WriteLine("DisposeAsync"); // valid pattern candidate
}
```

### 扩展方法

扩展方法**不能**用于实现异步释放。该方法必须是可访问的*实例*方法，才能被视为模式的有效候选。

### 可空值类型行为

_注意_：目前此功能未按规范工作，由 [#34701](https://github.com/dotnet/roslyn/issues/34701) 跟踪

对于 `using` 语句中的可空值类型，当前行为是仅当类型有值时才调用*底层*类型的 `Dispose`（即 `if(t.HasValue){ t.GetValueOrDefault().Dispose() }`）。这允许提升类型像底层类型一样使用，且仅在非 null 时调用 dispose。此行为扩展到 `await using` 语句中基于模式的 `DisposeAsync` 方法。

```csharp
public class C 
{
    public static async Task M()
    {
        StructDisposer? a = null;
        await using (a) { } // DisposeAsync is not invoked
        
        StructDisposer? b = new StructDisposer();
        await using (b) { } // DisposeAsync is invoked
    }
}

public struct StructDisposer
{
    public async ValueTask DisposeAsync() => Console.WriteLine("DisposeAsync");
}
```


## ref struct 的基于模式的释放

目前 `ref struct` 无法参与 `IDisposable` 模式，因为它们不能实现接口。此功能允许 `ref struct` 在满足特定结构要求时被视为可释放。具体要求为：

    "一个可访问的、返回 void 的实例方法，名为 Dispose，可以不带显式参数调用"

其中可访问意味着在正常可访问性规则下，从 `using(...)` 处合法调用。

```csharp
using System;
public class C 
{
    public void M()
    {
        using var x = new Disposer();
    }
}

ref struct Disposer
{
    internal void Dispose() => Console.WriteLine("Disposed");
}
```

_注意：_ `ref struct` 不能在 `async` 方法中使用，因此不能通过 `await using` 参与异步释放

### 可选参数和 params

基于模式的释放方法可以包含可选参数或 `params` 参数，并仍满足模式要求。一般而言，如果在 `using` 语法处能够以正常语言规则写出有效的 `s.Dispose()` 调用，则该 `ref struct` 被视为可释放。

```csharp
public ref struct Disposer
{
    public void Dispose(int x = 0, params object[] args) => Console.WriteLine("Disposed"); // valid pattern candidate
}
```

### 扩展方法

扩展方法**不能**用于实现释放。该方法必须是可访问的*实例*方法，才能被视为模式的有效候选。
