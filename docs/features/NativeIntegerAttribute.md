本机整数元数据
=========
以下描述了本机整数在元数据中的表示方式。

## NativeIntegerAttribute
类型引用在元数据中使用 `NativeIntegerAttribute` 进行注解。

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
    public sealed class NativeIntegerAttribute : Attribute
    {
        public NativeIntegerAttribute()
        {
            TransformFlags = new[] { true };
        }
        public NativeIntegerAttribute(bool[] flags)
        {
            TransformFlags = flags;
        }
        public readonly bool[] TransformFlags;
    }
}
```

`NativeIntegerAttribute` 类型仅供编译器使用——不允许在源代码中使用。
如果编译中尚未包含该类型声明，编译器将合成该类型。

元数据中的每个类型引用可以有一个关联的 `NativeIntegerAttribute`，其中包含一个 `bool[]`，每个 `bool` 表示关联类型是本机整数（C# 中的 `nint` 或 `nuint`）还是底层类型（`System.IntPtr` 或 `System.UIntPtr`）。
与类型引用关联的自定义修饰符会被忽略。

每个类型引用最多可以有一个 `NativeIntegerAttribute`。
该特性不被继承。

`bool[]` 的构造方式如下：
- `System.IntPtr`、`System.UIntPtr`：如果类型表示本机整数则为 `true`，否则为 `false`
- 泛型类型：按顺序包括含容器类型的类型参数的表示
- 元组：底层构造类型的表示
- 数组：元素类型的表示
- 指针类型：所指向类型的表示
- 函数指针类型：返回类型的表示，后跟按顺序排列的参数类型
- 其他：无

根据上述构造方式，`bool[]` 中的元素数量与类型中 `System.IntPtr` 和 `System.UIntPtr` 的引用数量相匹配。

## 优化

如果 `bool[]` 为空，则省略 `NativeIntegerAttribute`。

如果 `bool[]` 包含单个 `true` 值，则 `NativeIntegerAttribute` 使用无参数构造函数。

## 示例
```C#
System.IntPtr A;         // System.IntPtr A
nint B;                  // [NativeInteger] System.IntPtr B
IEnumerable<nuint> C     // [NativeInteger] System.Collections.Generic.IEnumerable<System.IntPtr> C
(System.IntPtr, nint) D; // [NativeInteger(new[] { false, true })] System.ValueTuple<System.IntPtr, System.IntPtr> D
```
