# 只读实例成员

提案 Issue：<https://github.com/dotnet/csharplang/issues/1710>

## 摘要

提供一种方式，可以为 struct 上的单个实例成员指定其不修改状态，类似于 `readonly struct` 指定所有实例成员都不修改状态的方式。

值得注意的是，`readonly instance member`（只读实例成员）不等于 `pure instance member`（纯实例成员）。`pure` 实例成员保证不会修改任何状态，而 `readonly` 实例成员仅保证不修改实例状态。

`readonly struct` 上的所有实例成员都隐式地是 `readonly instance members`。在非只读 struct 上显式声明的 `readonly instance members` 将以相同的方式运行。例如，如果你调用了一个本身不是只读的实例成员（在当前实例或实例的某个字段上），编译器仍然会创建隐藏副本。

## 设计

允许用户指定某个实例成员本身是 `readonly` 的，不会修改实例状态（当然需要编译器进行所有适当的验证）。例如：

```csharp
public struct Vector2
{
    public float x;
    public float y;

    public readonly float GetLengthReadonly()
    {
        return MathF.Sqrt(LengthSquared);
    }

    public float GetLength()
    {
        return MathF.Sqrt(LengthSquared);
    }

    public readonly float GetLengthIllegal()
    {
        var tmp = MathF.Sqrt(LengthSquared);

        x = tmp;    // Compiler error, cannot write x
        y = tmp;    // Compiler error, cannot write y

        return tmp;
    }

    public float LengthSquared
    {
        readonly get
        {
            return (x * x) +
                   (y * y);
        }
    }
}

public static class MyClass
{
    public static float ExistingBehavior(in Vector2 vector)
    {
        // This code causes a hidden copy, the compiler effectively emits:
        //    var tmpVector = vector;
        //    return tmpVector.GetLength();
        //
        // This is done because the compiler doesn't know that `GetLength()`
        // won't mutate `vector`.

        return vector.GetLength();
    }

    public static float ReadonlyBehavior(in Vector2 vector)
    {
        // This code is emitted exactly as listed. There are no hidden
        // copies as the `readonly` modifier indicates that the method
        // won't mutate `vector`.

        return vector.GetLengthReadonly();
    }
}
```

`readonly` 可以应用于属性访问器，以指示在访问器中不会修改 `this`。

```csharp
public int Prop
{
    readonly get
    {
        return this._prop1;
    }
}
```

当 `readonly` 应用于属性语法时，表示所有访问器均为 `readonly`。

```csharp
public readonly int Prop
{
    get
    {
        return this._store["Prop2"];
    }
    set
    {
        this._store["Prop2"] = value;
    }
}
```

与属性可访问性修饰符的规则类似，属性上不允许出现冗余的 `readonly` 修饰符。

```csharp
public readonly int Prop1 { readonly get => 42; } // Not allowed
public int Prop2 { readonly get => this._store["Prop2"]; readonly set => this._store["Prop2"]; } // Not allowed
```

`readonly` 只能应用于不修改包含类型的访问器。

```csharp
public int Prop
{
    readonly get
    {
        return this._prop3;
    }
    set
    {
        this._prop3 = value;
    }
}
```

### 自动属性
`readonly` 可以应用于自动实现的属性或 `get` 访问器。但是，编译器会将所有自动实现的 getter 视为只读，无论是否存在 `readonly` 修饰符。

```csharp
// Allowed
public readonly int Prop1 { get; }
public int Prop2 { readonly get; }
public int Prop3 { readonly get; set; }

// Not allowed
public readonly int Prop4 { get; set; }
public int Prop5 { get; readonly set; }
```

### 事件
`readonly` 可以应用于手动实现的事件，但不能应用于类字段式事件。`readonly` 不能应用于单个事件访问器（add/remove）。

```csharp
// Allowed
public readonly event Action<EventArgs> Event1
{
    add { }
    remove { }
}

// Not allowed
public readonly event Action<EventArgs> Event2;
public event Action<EventArgs> Event3
{
    readonly add { }
    remove { }
}
public static readonly event Event4
{
    add { }
    remove { }
}
```

其他一些语法示例：

* 表达式体成员：`public readonly float ExpressionBodiedMember => (x * x) + (y * y);`
* 泛型约束：`public readonly void GenericMethod<T>(T value) where T : struct { }`

编译器将像往常一样生成实例成员，并另外生成一个编译器识别的特性，以指示该实例成员不修改状态。这实际上使得隐藏的 `this` 参数从 `ref T` 变为 `in T`。

这使用户可以安全地调用该实例方法，而无需编译器创建副本。

一些明确允许的"边界"情况：

* 显式接口实现可以是 `readonly`。
* 分部方法可以是 `readonly`。两个签名必须都有或都没有 `readonly` 关键字。

限制包括：

* `readonly` 修饰符不能应用于静态方法、构造函数或析构函数。
* `readonly` 修饰符不能应用于委托。
* `readonly` 修饰符不能应用于类或接口的成员。

## 编译器 API

将添加以下公共 API：

- `bool IMethodSymbol.IsDeclaredReadOnly { get; }`：指示该成员因具有 readonly 关键字而为只读，或因为是 struct 上的自动实现实例 getter 而为只读。
- `bool IMethodSymbol.IsEffectivelyReadOnly { get; }`：指示该成员因 IsDeclaredReadOnly 为只读，或因包含类型为只读而为只读，但构造函数除外。

可能需要向属性和事件添加额外的公共 API。这对属性来说是个挑战，因为 `ReadOnly` 属性在 VB.NET 中已有不同的含义，并且 `IMethodSymbol.IsReadOnly` API 已经发布用于描述该场景。此特定问题在 https://github.com/dotnet/roslyn/issues/34213 中跟踪。
