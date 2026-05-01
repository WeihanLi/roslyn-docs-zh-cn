可空元数据
=========
以下描述了元数据中可空注解的表示方式。

## NullableAttribute
类型引用在元数据中用 `NullableAttribute` 进行注解。

```C#
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(
        AttributeTargets.Class |
        AttributeTargets.Event |
        AttributeTargets.Field |
        AttributeTargets.GenericParameter |
        AttributeTargets.Parameter |
        AttributeTargets.Property |
        AttributeTargets.ReturnValue,
        AllowMultiple = false,
        Inherited = false)]
    public sealed class NullableAttribute : Attribute
    {
        public readonly byte[] NullableFlags;
        public NullableAttribute(byte flag)
        {
            NullableFlags = new byte[] { flag };
        }
        public NullableAttribute(byte[] flags)
        {
            NullableFlags = flags;
        }
    }
}
```

`NullableAttribute` 类型仅供编译器使用——不允许在源代码中使用。
如果编译中尚未包含该类型声明，编译器会合成它。

元数据中的每个类型引用可能有一个关联的 `NullableAttribute`，其中包含一个 `byte[]`，每个 `byte` 表示可空性：0 表示忽略，1 表示未注解（不可空），2 表示已注解（可空）。

`byte[]` 的构造方式如下：
- 引用类型：可空性（0、1 或 2），后跟按顺序排列的类型参数（包括包含类型）的表示
- 可空值类型：仅为类型参数的表示
- 非泛型值类型：跳过
- 泛型值类型：0，后跟按顺序排列的类型参数（包括包含类型）的表示
- 数组：可空性（0、1 或 2），后跟元素类型的表示
- 元组：基础构造类型的表示
- 类型参数引用：可空性（0、1 或 2，对于无约束类型参数为 0）

注意，非泛型值类型由空的 `byte[]` 表示。
但是，泛型值类型和约束为值类型的类型参数在 `byte[]` 中对可空性有一个显式的 0。
泛型类型和类型参数用显式 `byte` 表示的原因是为了简化元数据导入。
具体来说，这避免了在解码可空性元数据时需要计算类型参数是否约束为值类型，因为约束可能包含对类型参数的（有效）循环引用。

### 优化

如果 `byte[]` 为空，则省略 `NullableAttribute`。

如果 `byte[]` 中所有值相同，则用该单个 `byte` 值构造 `NullableAttribute`。（例如，使用 `NullableAttribute(1)` 而不是 `NullableAttribute(new byte[] { 1, 1 })`。）

### 类型参数
每个类型参数定义可能有一个关联的 `NullableAttribute`，包含单个 `byte`：

1. `notnull` 约束：`NullableAttribute(1)`
2. `#nullable disable` 上下文中的 `class` 约束：`NullableAttribute(0)`
3. `#nullable enable` 上下文中的 `class` 约束：`NullableAttribute(1)`
4. `class?` 约束：`NullableAttribute(2)`
5. `#nullable disable` 上下文中无 `notnull`、`class`、`struct`、`unmanaged` 或类型约束：`NullableAttribute(0)`
6. `#nullable enable` 上下文中无 `notnull`、`class`、`struct`、`unmanaged` 或类型约束（等同于 `object?` 约束）：`NullableAttribute(2)`

## NullableContextAttribute
`NullableContextAttribute` 可用于指示没有 `NullableAttribute` 注解的类型引用的可空性。

```C#
namespace System.Runtime.CompilerServices
{
    [System.AttributeUsage(
        AttributeTargets.Class |
        AttributeTargets.Delegate |
        AttributeTargets.Interface |
        AttributeTargets.Method |
        AttributeTargets.Struct,
        AllowMultiple = false,
        Inherited = false)]
    public sealed class NullableContextAttribute : Attribute
    {
        public readonly byte Flag;
        public NullableContextAttribute(byte flag)
        {
            Flag = flag;
        }
    }
}
```

`NullableContextAttribute` 类型仅供编译器使用——不允许在源代码中使用。
如果编译中尚未包含该类型声明，编译器会合成它。

`NullableContextAttribute` 是可选的——仅使用 `NullableAttribute` 就可以完全准确地在元数据中表示可空注解。

`NullableContextAttribute` 在元数据中对类型和方法声明有效。
`byte` 值表示该作用域内没有显式 `NullableAttribute` 且不会由空的 `byte[]` 表示的类型引用的隐式 `NullableAttribute` 值。
元数据层级中最近的 `NullableContextAttribute` 适用。
如果层级中没有 `NullableContextAttribute` 特性，缺失的 `NullableAttribute` 特性被视为 `NullableAttribute(0)`。

该特性不被继承。

C#8 编译器使用以下算法来确定要发出哪些 `NullableAttribute` 和 `NullableContextAttribute` 特性。
首先，在每个类型引用和类型参数定义处生成 `NullableAttribute` 特性，方法是：计算 `byte[]`，跳过空的 `byte[]`，并在可能的情况下将 `byte[]` 折叠为单个 `byte`。
然后从方法开始，在元数据层级的每个级别：
编译器在该级别找到所有 `NullableAttribute` 特性以及直接子级的任何 `NullableContextAttribute` 特性中最常见的单个 `byte` 值。
如果没有单个 `byte` 值，则不做任何更改。
否则，在该级别创建 `NullableContext(value)` 特性，其中 `value` 是最常见的值（优先选择 `0` 而非 `1`，优先选择 `1` 而非 `2`），并删除所有具有该值的 `NullableAttribute` 和 `NullableContextAttribute` 特性。

注意，如果使用 C#8 编译的程序集中所有引用类型都是忽略状态，则不会发出任何 `NullableContextAttribute` 和 `NullableAttribute` 特性。
这等同于旧版程序集。

### 示例
```C#
// C# 元数据表示
[NullableContext(2)]
class Program
{
    string s;                        // string?
    [Nullable({ 2, 1, 2 }] Dictionary<string, object> d; // Dictionary<string!, object?>?
    [Nullable(1)] int[] a;           // int[]!
    int[] b;                         // int[]?
    [Nullable({ 0, 2 })] object[] c; // object?[]~
}
```

## 私有成员

为减少元数据大小，C#8 编译器可以配置为不为程序集外部不可访问的成员（`private` 成员，以及如果程序集不包含 `InternalsVisibleToAttribute` 特性，则还包括 `internal` 成员）发出特性。

编译器行为通过命令行标志进行配置。
目前使用特性标志：`-features:nullablePublicOnly`。

如果删除了私有成员特性，编译器将发出 `[module: NullablePublicOnly]` 特性。
`NullablePublicOnlyAttribute` 的存在或缺失可供工具用于解释没有关联 `NullableAttribute` 特性的私有成员的可空性。

对于元数据中没有显式可访问性的成员（特别是参数、类型参数、事件和属性），编译器使用容器的可访问性来确定是否发出可空特性。

```C#
namespace System.Runtime.CompilerServices
{
    [System.AttributeUsage(AttributeTargets.Module, AllowMultiple = false)]
    public sealed class NullablePublicOnlyAttribute : Attribute
    {
        public readonly bool IncludesInternals;
        public NullablePublicOnlyAttribute(bool includesInternals)
        {
            IncludesInternals = includesInternals;
        }
    }
}
```

`NullablePublicOnlyAttribute` 类型仅供编译器使用——不允许在源代码中使用。
如果编译中尚未包含该类型声明，编译器会合成它。

如果 `internal` 成员除 `public` 和 `protected` 成员之外也已注解，则 `IncludesInternal` 为 true。

## 兼容性

可空元数据不包含显式版本号。
在可能的情况下，编译器将静默忽略意外的特性形式。

此处描述的元数据格式与早期 C#8 预览版使用的格式不兼容：
1. 具体的非泛型值类型不再包含在 `byte[]` 中，以及
2. 使用 `NullableContextAttribute` 特性代替显式 `NullableAttribute` 特性。

这些差异意味着使用早期预览版编译的程序集可能被较新预览版错误读取，而使用较新预览版编译的程序集也可能被早期预览版错误读取。
