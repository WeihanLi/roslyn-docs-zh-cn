# 在公共方法中添加可选参数 #

Roslyn 对维护跨次要版本发布的源代码和二进制 API 兼容性有严格的要求。添加现有方法的重载可以有效地扩展公共 API，而不会破坏向后兼容性。但是，涉及可选参数的重载可能难以实现源代码和二进制兼容性。在向公共方法添加可选参数时，应遵循以下步骤。

## 向现有方法添加可选参数的步骤 ##

1.	检查是否已存在带有以下注释的方法
    
    ```
    // <上一版本> BACKCOMPAT OVERLOAD -- DO NOT TOUCH
    ```
2.	如果不存在：
    1. 复制该方法并将所有现有的可选参数设为必需参数。
    2. 更改方法体以调用原始方法。
    3. 添加注释 `// <上一版本> BACKCOMPAT OVERLOAD -- DO NOT TOUCH`。
       此方法为针对以前 API 编译的程序集提供二进制兼容性。
3.	将具有默认值的新参数添加到原始方法的*末尾*。
    这为以前 API 的使用者提供源代码兼容性。

## 示例 ##

之前：
```csharp
    public void O(string o1 = null, string o2 = null)
    {
    }
```

之后：

```csharp
    public void O(string o1 = null, string o2 = null, bool o3 = false)
    {
    }

    // 1.0 BACKCOMPAT OVERLOAD -- DO NOT TOUCH
    public void O(string o1, string o2)
    {
        O(o1, o2, o3: false);
    }
```

## 需要避免的事项 ##

**不要在现有可选参数中间添加参数**

```csharp
    public void O(string o1 = null, bool o4 = false, string o2 = null, bool o3 = false, bool o5 = false)
    {
    }
```

当跳过方法调用的某些参数时（如 `O(null, null, o5: true);`），这会破坏源代码兼容性。

**不要添加多个带有可选参数的重载**

```csharp
    public void O(string o1 = null, string o2 = null, bool o3 = false)
    {
    }

    // 1.0 BACKCOMPAT OVERLOAD -- DO NOT TOUCH
    public void O(string o1 = null, string o2 = null)
    {
         O(o1, o2, o3: false);
    }
```

除了这通常是不明确的而且无法编译之外，它还使验证兼容性变得更加困难。

## 最终结果 ##

生成的代码应该：1）只有一个带有可选参数的重载，2）该重载在所有重载中具有最多的参数，3）所有以前的重载仍应存在，正确注释其发布版本，并且只包含必需参数。

## 注意 ##

在此更改之后，如果 Public API Analyzer 报告问题，您应该从 PublicAPI.Shipped.txt 复制更改的条目，然后将该条目放入 PublicAPI.Unshipped.txt 中，并加上 `*REMOVED*` 前缀。

PublicAPI.Shipped.txt

``` txt
Example.O(string o1 = null, string o2 = null) -> void
```

PublicAPI.Unshipped.txt

``` txt
*REMOVED*Example.O(string o1 = null, string o2 = null) -> void
```
