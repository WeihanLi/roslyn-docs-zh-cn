ExperimentalAttribute
=====================
对标有 `Windows.Foundation.Metadata.ExperimentalAttribute` 的类型和成员的引用报告警告。
```
namespace Windows.Foundation.Metadata
{
    [AttributeUsage(
        AttributeTargets.Class | AttributeTargets.Struct | AttributeTargets.Enum |
        AttributeTargets.Interface | AttributeTargets.Delegate,
        AllowMultiple = false)]
    public sealed class ExperimentalAttribute : Attribute
    {
    }
}
```

## 警告
警告消息为特定消息，其中 `'{0}'` 是完全限定的类型或成员名称。
```
'{0}' is for evaluation purposes only and is subject to change or removal in future updates.
```

对标有该特性的类型或成员的任何引用都会报告此警告。
（`AttributeUsage` 仅适用于类型，因此该特性无法从 C# 或 VB 中应用于非类型成员，
但可以从 C# 或 VB 外部将该特性应用于非类型成员。）
```
[Experimental]
class A
{
    internal object F;
    internal static object G;
    internal class B { }
}
class C
{
    static void Main()
    {
        F(default(A));   // warning CS08305: 'A' is for evaluation purposes only ...
        F(typeof(A));    // warning CS08305: 'A' is for evaluation purposes only ...
        F(nameof(A));    // warning CS08305: 'A' is for evaluation purposes only ...
        var a = new A(); // warning CS08305: 'A' is for evaluation purposes only ...
        F(a.F);
        F(A.G);          // warning CS08305: 'A' is for evaluation purposes only ...
        F<A.B>(null);    // warning CS08305: 'A' is for evaluation purposes only ...
    }
    static void F<T>(T t)
    {
    }
}
```

该特性不会从基类型或重写成员继承。
```
[Experimental]
class A<T>
{
    [Experimental]
    internal virtual void F()
    {
    }
}
class B : A<int> // warning CS08305: 'A<int>' is for evaluation purposes only ...
{
    internal override void F()
    {
        base.F(); // warning CS08305: 'A<int>.F' is for evaluation purposes only ...
    }
}
class C
{
    static void F(A<object> a, B b) // warning CS08305: 'A<object>' is for evaluation purposes only ...
    {
        a.F(); // warning CS08305: 'A<object>.F' is for evaluation purposes only ...
        b.F();
    }
}
```

不会自动抑制警告：即使在 `[Experimental]` 成员内部，所有引用也会生成警告。
抑制警告需要显式的编译器选项或 `#pragma`。
```
[Experimental] enum E { }
[Experimental]
class C
{
    private C(E e) // warning CS08305: 'E' is for evaluation purposes only ...
    {
    }
    internal static C Create() // warning CS08305: 'C' is for evaluation purposes only ...
    {
        return Create(0);
    }
#pragma warning disable 8305
    internal static C Create(E e)
    {
        return new C(e);
    }
}
```

## ObsoleteAttribute 和 DeprecatedAttribute
`ExperimentalAttribute` 与 `System.ObsoleteAttribute` 或 `Windows.Framework.Metadata.DeprecatedAttribute` 相互独立。

`[Experimental]` 的警告会在 `[Obsolete]` 或 `[Deprecated]` 成员内部报告。
`[Obsolete]` 和 `[Deprecated]` 的警告和错误会在 `[Experimental]` 成员内部报告。
```
[Obsolete]
class A
{
    static object F() => new C(); // warning CS08305: 'C' is for evaluation purposes only ...
}
[Deprecated(null, DeprecationType.Deprecate, 0)]
class B
{
    static object F() => new C(); // warning CS08305: 'C' is for evaluation purposes only ...
}
[Experimental]
class C
{
    static object F() => new B(); // warning CS0612: 'B' is obsolete
}
```

如果同时存在多个特性，则优先报告 `[Obsolete]` 和 `[Deprecated]` 的警告和错误，而不是 `[Experimental]` 的警告。
```
[Obsolete]
[Experimental]
class A
{
}
[Experimental]
[Deprecated(null, DeprecationType.Deprecate, 0)]
class B
{
}
class C
{
    static A F() => null; // warning CS0612: 'A' is obsolete
    static B G() => null; // warning CS0612: 'B' is obsolete
}

```
