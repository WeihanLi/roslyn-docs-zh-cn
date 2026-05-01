可空引用类型
=========
引用类型可以是可空的、不可空的或空忽略的（此处分别缩写为 `?`、`!` 和 `~`）。

## 设置项目级可空上下文

可以使用 "nullable" 命令行开关来设置项目级可空上下文：
-nullable[+|-]                        指定可空上下文选项 enable|disable。
-nullable:{enable|disable|warnings|annotations}   指定可空上下文选项 enable|disable|warnings|annotations。

通过 msbuild，可以为 Csc 构建任务的 "Nullable" 参数提供参数来设置上下文。
接受的值为 "enable"、"disable"、"warnings"、"annotations" 或 null（表示编译器的默认可空上下文）。
Microsoft.CSharp.Core.targets 会将名为 "Nullable" 的 msbuild 属性的值传递给该参数。

注意，在 C# 8.0 的早期预览版本中，此 "Nullable" 属性先后被命名为 "NullableReferenceTypes"，然后是 "NullableContextOptions"。

## 注解
在源代码中，可空引用类型用 `?` 注解。
```c#
string? OptString; // 可以为 null
Dictionary<string, object?>? OptDictionaryOptValues; // 字典可以为 null，值也可以为 null
```
在 `#nullable` 上下文之外对引用类型用 `?` 注解时会报警告。

[元数据表示](nullable-metadata.md)

## 声明警告
_描述初始绑定中为声明报告的警告。_

## 流分析
流分析用于推断可执行代码中变量的可空性。变量的推断可空性独立于变量声明的可空性。
即使方法调用在条件上被省略，也会对其进行分析。例如，在发布模式下的 `Debug.Assert`。

### 警告
_描述警告集合。区分 W 警告。_
如果分析确定 null 检查始终（或从不）通过，则会生成隐藏的诊断信息。例如：`"string" is null`。

### Null 测试
以下几种 null 检查会在测试时影响流状态：
- 与 `null` 的比较：`x == null` 和 `x != null`
- `is` 运算符：`x is null`、`x is K`（其中 `K` 是常量）、`x is string`、`x is string s`
- 对众所周知的相等方法的调用，包括：
  - `static bool object.Equals(object, object)`
  - `static bool object.ReferenceEquals(object, object)`
  - `bool object.Equals(object)` 及其重写
  - `bool IEquatable<T>.Equals(T)` 及其实现
  - `bool IEqualityComparer<T>.Equals(T, T)` 及其实现

某些 null 检查是"纯 null 测试"，这意味着它们可能导致之前流状态为非空的变量更新为可能为空。纯 null 测试包括：
- `x == null`、`x != null` *无论使用内置运算符还是用户定义运算符*
- `(Type)x == null`、`(Type)x != null`
- `x is null`
- `object.Equals(x, null)`、`object.ReferenceEquals(x, null)`
- `IEqualityComparer<Type?>.Equals(x, null)`

除 `x is null` 外，以上所有检查都是可交换的。例如，`null == x` 也是有效的纯 null 测试。

某些可能不返回 `bool` 的表达式也被视为纯 null 测试：
- `x?.Member` 会无条件地将接收者的流状态更改为可能为空
- `x ?? y` 会无条件地将左侧表达式的流状态更改为可能为空

纯 null 测试如何影响流分析的示例：
```cs
string s = "hello";
if (s != null)
{
    _ = s.ToString(); // ok
}
else
{
    _ = s.ToString(); // warning
}
```

与"非纯" null 测试的对比：
```cs
string s = "hello";
if (s is string)
{
    _ = s.ToString(); // ok
}
else
{
    _ = s.ToString(); // ok
}
```

调用使用以下特性注解的方法也会影响流分析：
- 简单前置条件：`[AllowNull]` 和 `[DisallowNull]`
- 简单后置条件：`[MaybeNull]` 和 `[NotNull]`
- 条件后置条件：`[MaybeNullWhen(bool)]` 和 `[NotNullWhen(bool)]`
- `[DoesNotReturnIf(bool)]`（例如 `[DoesNotReturnIf(false)]` 用于 `Debug.Assert`）和 `[DoesNotReturn]`
- `[NotNullIfNotNull(string)]`
 - 成员后置条件：`[MemberNotNull(params string[])]` 和 `[MemberNotNullWhen(bool, params string[])]`
参见 https://github.com/dotnet/csharplang/blob/main/meetings/2019/LDM-2019-05-15.md

`Interlocked.CompareExchange` 方法由于其可空性语义的复杂性，在流分析中有特殊处理，而不是通过注解来处理。受影响的重载包括：
- `static object? System.Threading.Interlocked.CompareExchange(ref object? location, object? value, object? comparand)`
- `static T System.Threading.Interlocked.CompareExchange<T>(ref T location, T value, T comparand) where T : class?`

当简单前置条件和后置条件特性应用于属性时，如果将允许的输入状态分配给该属性，属性的状态将更新为允许的输出状态。例如，`[AllowNull] string` 属性可以被赋值为 `null`，但仍然返回非空值。


在方法体内强制执行可空性特性：
- 用 `[AllowNull]` 标记的输入参数以可能为空（或可能为默认值）的状态初始化。
- 用 `[DisallowNull]` 标记的输入参数以非空状态初始化。
- 用 `[MaybeNull]` 或 `[MaybeNullWhen]` 标记的参数可以被赋予可能为空的值，不会产生警告。返回值同理。用 `[NotNullWhen]` 标记的可空参数同理（该特性被忽略）。
- 用 `[NotNull]` 标记的参数在被赋予可能为空的值时会产生警告。返回值同理。
- 用 `[MaybeNullWhen]`/`[NotNullWhen]` 标记的参数的状态会在退出方法时检查。
- 用 `[DoesNotReturn]` 标记的方法如果正常返回或退出则会产生警告。

注意：我们不验证自动属性的内部一致性，因此可能会像在字段上一样错误使用自动属性的特性。例如：`[AllowNull, NotNull] public TOpen P { get; set; }`。

在重写和实现时强制执行可空性特性：
除了检查重写/实现中的类型与被重写/实现成员兼容外，我们还检查可空性特性是否兼容。
- 对于输入参数（按值和 `in`），我们检查被重写/实现参数所允许的最宽泛值是否可以赋值给重写/实现参数。
- 对于输出参数（`out` 和返回值），我们检查重写/实现参数所产生的最宽泛值是否可以赋值给被重写/实现参数。
- 对于 `ref` 参数和返回值，两者都检查。
- 我们检查输入参数上的后置条件合同 `[NotNull]`/`[MaybeNull]` 是否由重写/实现成员执行。
- 我们检查如果被重写/实现成员具有 `[DoesNotReturn]` 特性，重写/实现是否也具有该特性。
- 我们检查 `[MemberNotNull(...)]` 或 `[MemberNotNullWhen(...)]` 中使用的成员在退出时不为空，方法是假设它们在进入时具有可能为空的状态。

## `default`
如果 `T` 是引用类型，则 `default(T)` 是 `T?`。
```c#
string? s = default(string); // 赋值 ?，无警告
string t = default; // 赋值 ?，有警告
```
如果 `T` 是值类型，则 `default(T)` 是 `T`，且 `T` 中任何非值类型字段可能为空。
如果 `T` 是值类型，则 `new T()` 等同于 `default(T)`。
```c#
struct Pair<T, U> { public T First; public U Second; }
var p = default(Pair<object?, string>); // ok: Pair<object?, string!> p
p.Second.ToString(); // warning
(object?, string) t = default; // ok
t.Item2.ToString(); // warning
```
如果 `T` 是无约束类型参数，则 `default(T)` 是一个可能为空的 `T`。
```c#
T t = default; // warning
```

### 转换
转换可以将 `~` 视为与 `?` 和 `!` 不同，也可以将 `~` 视为隐式可转换为 `?` 和 `!`。
给定 `IIn<in T>` 和 `IOut<out T>`，在 `~` 不同的情况下：
- `T!` 是 `T~` 是 `T?`
- `IIn<T!>` 是 `IIn<T~>` 是 `IIn<T?>`
- `IOut<T?>` 是 `IOut<T~>` 是 `IOut<T!>`
大多数转换将 `~` 视为隐式可转换为 `?` 和 `!`。

_描述来自用户定义转换的警告。_

### 赋值
对于 `x = y`，`y` 的转换类型的可空性用于 `x`。
如果比较 `x` 的推断可空性和 `y` 的声明类型时发现顶层或嵌套可空性不匹配，则报警告。
当赋值 `?` 给 `!` 且目标是局部变量时，该警告为 W 警告。
```c#
notNull = maybeNull; // 赋值 ?，有警告
notNull = oblivious; // 赋值 ~，无警告
oblivious = maybeNull; // 赋值 ?，无警告
```

### 局部变量声明
可空性遵循上述赋值规则。将 `?` 赋值给 `!` 是 W 警告。
```c#
string notNull = maybeNull; // 赋值 ?，有警告
```
`var` 声明的可空性是推断类型的可空版本。
```c#
var s = maybeNull; // s 是 ?，无警告
if (maybeNull == null) return;
var t = maybeNull; // t 也是 ?
```

### 抑制运算符（`!`）
后缀 `!` 运算符将顶层可空性设置为不可空。对抑制表达式的转换不产生可空性警告。
```c#
var x = optionalString!; // x 是 string!
var y = obliviousString!; // y 是 string!
var z = new [] { optionalString, obliviousString }!; // 无变化，z 是 string?[]!
```
在没有 `NonNullTypes` 上下文的情况下使用 `!` 运算符时会报警告。

不必要地使用 `!` 不会产生任何诊断，包括 `!!`。

如果操作数表达式 `e` 可以是目标类型，则抑制表达式 `e!` 也可以是目标类型。

### 显式转换
显式转换为 `?` 会更改顶层可空性。
显式转换为 `!` 不会更改顶层可空性，但可能产生 W 警告。
```c#
var x = (string)maybeNull; // x 是 string?，W 警告
var y = (string)oblivious; // y 是 string~，无警告
var z = (string?)notNull; // y 是 string?，无警告
```
如果操作数的类型（包括嵌套可空性）不能显式转换为目标类型，则报警告。
```c#
sealed class MyList<T> : IEnumerable<T> { }
var x =  new MyList<string?>();
var y = (IEnumerable<object>)x;  // warning
var z = (IEnumerable<object?>)x; // no warning
```

### 方法类型推断

我们修改了[固定](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#fixing "Fixing")的规范规则，以考虑可能等价（即具有恒等转换）但可能不完全相同的类型的边界集合。现有规范说（第三条）

> 如果在剩余候选类型 `Uj` 中存在唯一一个类型 `V`，从该类型存在到所有其他候选类型的隐式转换，则 `Xi` 固定为 `V`。

这对于 C# 5 来说不正确（即这是语言规范中的一个错误），因为它无法处理如 `Dictionary<object, dynamic>` 和 `Dictionary<dynamic, object>` 这样的边界，它们会合并为 `Dictionary<dynamic, dynamic>`。这是[一个未解决的问题](https://github.com/ECMA-TC49-TG2/spec/issues/951)，我们预计它将在 ECMA 规范的下一次迭代中得到解决。正确处理可空性将需要超出该下一次 ECMA 规范迭代的额外更改。

向类型参数的精确边界集合添加时，等价但不完全相同的类型使用*不变*规则合并。向类型参数的下界集合添加时，等价但不完全相同的类型使用*协变*规则合并。向类型参数的上界集合添加时，等价但不完全相同的类型使用*逆变*规则合并。在最终固定步骤中，等价但不完全相同的类型使用*协变*规则合并。

合并等价但不完全相同的类型的方法如下：

#### 不变合并规则

- 合并 `dynamic` 和 `object` 得到类型 `dynamic`。
- 在元素名称不同的元组类型的合并在其他地方指定。
- 合并可空性不同的等价类型的方法如下：合并类型 `Tn` 和 `Um`（其中 `n` 和 `m` 是不同的可空性注解）得到类型 `Vk`，其中 `V` 是使用不变规则合并 `T` 和 `U` 的结果，`k` 如下：
 - 如果 `n` 或 `m` 中任一为不可空，则为不可空。
 - 如果 `n` 或 `m` 中任一为可空，则为可空。
 - 否则为忽略。
- 合并构造泛型类型的方法如下：合并类型 `K<A1, A2, ...>` 和 `K<B1, B2, ...>` 得到类型 `K<C1, C2, ...>`，其中 `Ci` 是使用不变规则合并 `Ai` 和 `Bi` 的结果。
- 合并元组类型 `(A1, A2, ...)` 和 `(B1, B2, ...)` 得到类型 `(C1, C2, ...)`，其中 `Ci` 是使用不变规则合并 `Ai` 和 `Bi` 的结果。
- 合并数组类型 `T[]` 和 `U[]` 得到类型 `V[]`，其中 `V` 是使用不变规则合并 `T` 和 `U` 的结果。

#### 协变合并规则

- 合并 `dynamic` 和 `object` 得到类型 `dynamic`。
- 在元素名称不同的元组类型的合并在其他地方指定。
- 合并可空性不同的等价类型的方法如下：合并类型 `Tn` 和 `Um`（其中 `n` 和 `m` 是不同的可空性注解）得到类型 `Vk`，其中 `V` 是使用协变规则合并 `T` 和 `U` 的结果，`k` 如下：
 - 如果 `n` 或 `m` 中任一为可空，则为可空。
 - 如果 `n` 或 `m` 中任一为忽略，则为忽略。
 - 否则为不可空。
- 合并构造泛型类型的方法如下：合并类型 `K<A1, A2, ...>` 和 `K<B1, B2, ...>` 得到类型 `K<C1, C2, ...>`，其中 `Ci` 是通过以下规则合并 `Ai` 和 `Bi` 的结果：
  - 如果 `K` 中第 `i` 个位置的类型参数是不变的，使用不变规则。
  - 如果 `K` 中第 `i` 个位置的类型参数声明为 `out`，使用协变规则。
  - 如果 `K` 中第 `i` 个位置的类型参数声明为 `in`，使用逆变规则。
- 合并元组类型 `(A1, A2, ...)` 和 `(B1, B2, ...)` 得到类型 `(C1, C2, ...)`，其中 `Ci` 是使用协变规则合并 `Ai` 和 `Bi` 的结果。
- 合并数组类型 `T[]` 和 `U[]` 得到类型 `V[]`，其中 `V` 是使用不变规则合并 `T` 和 `U` 的结果。

#### 逆变合并规则

- 合并 `dynamic` 和 `object` 得到类型 `dynamic`。
- 在元素名称不同的元组类型的合并在其他地方指定。
- 合并可空性不同的等价类型的方法如下：合并类型 `Tn` 和 `Um`（其中 `n` 和 `m` 是不同的可空性注解）得到类型 `Vk`，其中 `V` 是使用逆变规则合并 `T` 和 `U` 的结果，`k` 如下：
 - 如果 `n` 或 `m` 中任一为不可空，则为不可空。
 - 如果 `n` 或 `m` 中任一为忽略，则为忽略。
 - 否则为可空。
- 合并构造泛型类型的方法如下：合并类型 `K<A1, A2, ...>` 和 `K<B1, B2, ...>` 得到类型 `K<C1, C2, ...>`，其中 `Ci` 是通过以下规则合并 `Ai` 和 `Bi` 的结果：
  - 如果 `K` 中第 `i` 个位置的类型参数是不变的，使用不变规则。
  - 如果 `K` 中第 `i` 个位置的类型参数声明为 `in`，使用协变规则。
  - 如果 `K` 中第 `i` 个位置的类型参数声明为 `out`，使用逆变规则。
- 合并元组类型 `(A1, A2, ...)` 和 `(B1, B2, ...)` 得到类型 `(C1, C2, ...)`，其中 `Ci` 是使用逆变规则合并 `Ai` 和 `Bi` 的结果。
- 合并数组类型 `T[]` 和 `U[]` 得到类型 `V[]`，其中 `V` 是使用不变规则合并 `T` 和 `U` 的结果。

这些合并规则的意图是具有结合律和交换律，因此编译器可以按任意顺序逐对合并一组等价类型以计算最终结果。

> ***未解决问题***：这些规则没有描述嵌套泛型类型 `K<A>.L<B>` 与 `K<C>.L<D>` 的合并处理。这应该与假设类型 `KL<A, B>` 与 `KL<C, D>` 的合并处理方式相同。

> ***未解决问题***：这些规则没有描述指针类型合并的处理。

### 数组创建
_最佳类型_元素可空性的计算使用上述转换规则和协变合并规则。
```c#
var w = new [] { notNull, oblivious }; // ~[]!
var x = new [] { notNull, maybeNull, oblivious }; // ?[]!
var y = new [] { enumerableOfNotNull, enumerableOfMaybeNull, enumerableOfOblivious }; // IEnumerable<?>!
var z = new [] { listOfNotNull, listOfMaybeNull, listOfOblivious }; // List<~>!
```

### 匿名类型
匿名类型字段的可空性由参数的可空性决定，从初始化表达式推断。
```c#
static void F<T>(T x, T y)
{
    if (x == null) return;
    var a = new { x, y }; // 推断为 x:T, y:T（均未注解）
    a.x.ToString(); // ok（通过属性 x 初始值的可空流状态跟踪非空）
    a.y.ToString(); // warning
}
```

### Null 合并运算符
`x ?? y` 的顶层可空性：如果 `x` 是 `!` 则为 `!`，否则为 `y` 的顶层可空性。
如果 `x` 和 `y` 之间存在嵌套可空性不匹配，则报警告。

### Try-finally
我们通过跟踪哪些变量可能在 finally 块中被赋予了 null 值来推断 try 语句之后的状态。
这种跟踪区分实际赋值和推断：
只有实际赋值（而非推断）会影响 try 语句之后的状态。
参见 https://github.com/dotnet/roslyn/issues/34018 和 https://github.com/dotnet/roslyn/pull/35276。
这尚未反映在可空引用类型的语言规范中（因为我们目前没有关于如何处理 try 语句的规范）。

## 类型参数
允许使用 `class?` 约束，该约束与 class 一样要求类型参数为引用类型，但允许它是可空的。
[可空草案](https://github.com/dotnet/csharplang/issues/790)
[4/25/18](https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-04-25.md)

不允许任何可空性的显式 `object`（或 `System.Object`）约束。但是，类型替换可能会导致 `object!` 或 `object~` 约束出现在约束类型中，当与其他约束相比其可空性有意义时。`object!` 约束要求类型为不可空（值类型或引用类型）。
但是，不允许显式 `object?` 约束。
无约束（这里指没有类型约束，也没有 `class`、`struct` 或 `unmanaged` 约束）的类型参数，在启用可空注解的上下文中声明时，本质上等同于约束为 `object?` 的类型参数。如果禁用注解，则类型参数本质上等同于约束为 `object~` 的类型参数。上下文由类型参数列表中声明类型参数的标识符处确定。
[4/25/18](https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-04-25.md)
注意，`object`/`System.Object` 约束在元数据中表示为任何其他类型约束，类型为 System.Object。

允许显式 `notnull` 约束，该约束要求类型为不可空（值类型或引用类型）。
[5/15/19](https://github.com/dotnet/csharplang/blob/main/meetings/2019/LDM-2019-05-15.md)
确定它是命名类型约束还是特殊 `notnull` 约束的规则类似于 `unmanaged` 的规则。同样，它只在约束列表的第一个位置有效。

对于具有 `class` 约束或不可空引用类型或接口类型约束的类型参数，使用可空类型参数时会报警告。
[4/25/18](https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-04-25.md)
```c#
static void F1<T>() where T : class { }
static void F2<T>() where T : Stream { }
static void F3<T>() where T : IDisposable { }
F1<Stream?>(); // warning
F2<Stream?>(); // warning
F3<Stream?>(); // warning
```
类型参数约束可以包含可空引用类型和接口类型。
[4/25/18](https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-04-25.md)
```c#
static void F2<T> where T : Stream? { }
static void F3<T>() where T : IDisposable? { }
F2<Stream?>(); // ok
F3<Stream?>(); // ok
```
约束类型顶层可空性不一致时会报警告。
[4/25/18](https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-04-25.md)
```c#
static void F4<T> where T : class, Stream? { } // warning
static void F5<T> where T : Stream?, IDisposable { } // warning
```
对于重复约束（在忽略顶层和嵌套可空性的情况下进行比较）会报错误。
```c#
class C<T> where T : class
{
    static void F1<U>() where U : T, T? { } // error: duplicate constraint
    static void F2<U>() where U : I<T>, I<T?> { } // error: duplicate constraint
}
```
_来自未注解（已注解）类型和方法的泛型类型参数的已注解（未注解）类型参数的规则是什么？_
```c#
[NotNullTypes(false)] List<T> F1<T>(T t) where T : class { ... }
[NotNullTypes(true)]  List<T> F2<T>(T t) where T : class { ... }
[NotNullTypes(true)]  List<T?> F3<T>(T? t) where T : class { ... }
var x = F1(notNullString);   // List<string!> 还是 List<string~> ？
var y = F1(maybeNullString); // List<string?> 还是 List<string~> ？
var z = F2(obliviousString); // List<string~>! 还是 List<string!>! ？
var w = F3(obliviousString); // List<string~>! 还是 List<string?>! ？
```
对于解引用类型为 T 的变量（其中 T 是无约束类型参数）会报警告。
```C#
static void F<T>(T t) => t.ToString(); // 警告：可能为 null 的解引用
```
当尝试将 null、default 或可能为 null 的值转换为无约束类型参数时会报警告。
```C#
static T F<T>() => default; // 警告：将 default 转换为 T
```

## 对象创建
为可空引用类型创建实例时会报错误。
```c#
new C?(); // error
new List<C?>(); // ok
```

## 生成的代码
旧版代码生成策略可能不支持可空感知。
将项目级可空上下文设置为"enable"可能会导致用户无法修复的许多警告。
为支持此场景，任何被确定为生成的语法树都会将其可空状态隐式设置为"disable"，无论整体项目状态如何。

符合以下一个或多个条件的语法树被确定为生成的：
- 文件名以以下内容开头：
    - TemporaryGeneratedFile_
- 文件名以以下内容结尾：
    - .designer.cs
    - .generated.cs
    - .g.cs
    - .g.i.cs
- 包含含有以下内容的顶层注释：
    - `<autogenerated`
    - `<auto-generated`

新版支持可空感知的生成器可以通过在代码开头包含生成的 `#nullable restore` 来选择启用可空分析，确保生成的语法树将根据项目级可空上下文进行分析。

## 公共 API
API 使用者可能想要回答以下几个问题：
1. 我应该在类型后面打印 `?` 吗？
2. 我可以将 `null` 值赋给此类型的变量吗？
3. 我能从此类型的变量读取到 `null` 值吗？

我们希望公开的两个基本概念是：`IsAnnotated` 和 `NonNullTypes`。
_我们也可能公开一些更高层次的概念（以方便回答问题 2 和 3）。_

|  | IsAnnotated |
|--| ----------- |
| `string?` | `true` |
| `int?` / `Nullable<int>` | `true`（_需要确认_） |
| `Nullable<T>` | `true` |
| `T? where T : class` | `true` |
| `T? where T : struct` | `true`（_需要确认_） |
| `string` | `false` |
| `int` | `false` |
| `T where T : class/object` | `false` |
| `T where T : class?` | `false` |
| `T where T : struct` | `false` |
| `T` | `false` |
