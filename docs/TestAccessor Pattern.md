# `TestAccessor` 模式

`TestAccessor` 模式允许生产代码为测试目的公开内部功能，而不会将内部功能提供给其他生产代码。该模式有两个主要组件：

1. `TestAccessor` 类型，包含仅用于测试的功能
2. `GetTestAccessor()` 方法，返回 `TestAccessor` 实例

该模式依赖于强制执行一个简单规则：不允许生产代码调用 `GetTestAccessor()` 方法。这可以通过代码审查或分析器来强制执行。与替代方案相比，此模式具有许多优势：

* 该模式不需要为测试目的扩展可访问性（例如 `internal` 而不是 `private`）
* 该模式是自文档化的：`TestAccessor` 类型中的所有属性和方法仅供测试代码使用
* 该模式足够一致，可以通过静态分析（分析器）强制执行
* 该模式足够简单，可以手动强制执行（代码审查）

## `TestAccessor` 类型

`TestAccessor` 类型通常定义为嵌套结构。在以下示例中，`SomeProductionType.TestAccessor.PrivateStateData` 属性允许测试代码读取和写入私有字段 `SomeProductionType._privateStateData` 的值，而不会将 `_privateStateData` 字段暴露给其他生产代码。

```csharp
internal class SomeProductionType
{
  private int _privateStateData;

  internal readonly struct TestAccessor
  {
    private readonly SomeProductionType _instance;

    internal TestAccessor(SomeProductionType instance)
    {
      _instance = instance;
    }

    internal ref int PrivateStateData => ref _instance._privateStateData;
  }
}
```

## `GetTestAccessor()` 方法

`GetTestAccessor()` 方法始终定义如下：

```csharp
internal TestAccessor GetTestAccessor()
  => new TestAccessor(this);
```
