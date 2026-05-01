# 拦截器

## 摘要

*拦截器*是一种 C# 编译器功能，最初在 .NET 8 中以实验性方式发布，在 .NET 9.0.2xx SDK 及更高版本中提供稳定支持。

*拦截器*是一种方法，可以在编译时通过声明的方式将对*可拦截*方法的调用替换为对自身的调用。这种替换通过让拦截器声明其拦截的调用的源位置来实现。这提供了一种有限的功能，可以通过向编译中添加新代码（例如在源生成器中）来更改现有代码的语义。

```cs
using System;
using System.Runtime.CompilerServices;

var c = new C();
c.InterceptableMethod(1); // L1: 打印 "interceptor 1"
c.InterceptableMethod(1); // L2: 打印 "other interceptor 1"
c.InterceptableMethod(2); // L3: 打印 "other interceptor 2"
c.InterceptableMethod(1); // 打印 "interceptable 1"

class C
{
    public void InterceptableMethod(int param)
    {
        Console.WriteLine($"interceptable {param}");
    }
}

// generated code
static class D
{
    [InterceptsLocation(version: 1, data: "...(refers to the call at L1)")]
    public static void InterceptorMethod(this C c, int param)
    {
        Console.WriteLine($"interceptor {param}");
    }

    [InterceptsLocation(version: 1, data: "...(refers to the call at L2)")]
    [InterceptsLocation(version: 1, data: "...(refers to the call at L3)")]
    public static void OtherInterceptorMethod(this C c, int param)
    {
        Console.WriteLine($"other interceptor {param}");
    }
}
```

## 详细设计

### InterceptsLocationAttribute

一个方法通过添加一个或多个 `[InterceptsLocation]` 特性来表明它是一个*拦截器*。这些特性引用其所拦截调用的源位置。

```cs
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    public sealed class InterceptsLocationAttribute(int version, string data) : Attribute
    {
    }
}
```

任何"普通方法"（即 `MethodKind.Ordinary`）都可以拦截其调用。

除了普通形式 `M()` 和 `receiver.M()` 之外，条件访问中的调用（例如 `receiver?.M()` 形式）也可以被拦截。接收者是指针成员访问的调用（例如 `ptr->M()` 形式）也可以被拦截。

源代码中包含的 `[InterceptsLocation]` 特性会与其他自定义特性一样发出到生成的程序集中。

此类型的文件本地声明（`file class InterceptsLocationAttribute`）是有效的，当它们位于同一文件和编译中时，编译器会识别其用法。需要声明此特性的生成器应使用文件本地声明，以确保不与其他需要执行相同操作的生成器发生冲突。

#### 位置编码

`[InterceptsLocation]` 的参数为：
1. 版本号。编译器将来可能会引入新的位置编码，并对应新的版本号。
2. 不透明的数据字符串。这不打算供人类阅读。

"版本 1"的数据编码是一个 base64 编码的字符串，包含以下数据：
- 包含被拦截调用的文件的 16 字节 xxHash128 内容校验和。
- 调用在语法中的位置（即 `SyntaxNode.Position`）的 int32 小端格式。
- utf-8 字符串数据，包含用于错误报告的显示文件名。

#### 位置

调用的位置是表示可拦截方法的简单名称语法的位置。例如，在 `app.MapGet(...)` 中，`MapGet` 的名称语法将被视为调用的位置。对于像 `System.Console.WriteLine(...)` 这样的静态方法调用，`WriteLine` 的名称语法是调用的位置。如果将来允许拦截对属性访问器的调用（例如 `obj.Property`），我们也可以使用名称语法。

#### 特性创建

Roslyn 提供了 API `GetInterceptableLocation(this SemanticModel, InvocationExpressionSyntax, CancellationToken)`，用于将 `[InterceptsLocation]` 插入生成的源代码中。我们建议源生成器依赖此 API 来拦截调用。详见 https://github.com/dotnet/roslyn/issues/72133。

### 非调用方法用法

转换为委托类型、取地址等方法用法不能被拦截。

拦截只能针对对普通成员方法的调用——不包括构造函数、委托、属性、本地函数、运算符等。将来可能会添加对更多成员类型的支持。

### 元数

拦截器不能在任何嵌套级别的泛型类型中声明。

拦截器必须是非泛型的，或者其元数等于原始方法的元数与包含类型元数之和。例如：

```cs
Grandparent<int>.Parent<bool>.Original<string>(1, false, "a"); // L1

class Grandparent<T1>
{
    class Parent<T2>
    {
        public static void Original<T3>(T1 t1, T2 t2, T3 t3) { }
    }
}

class Interceptors
{
    [InterceptsLocation(1, "..(refers to call at L1)")]
    public static void Interceptor<T1, T2, T3>(T1 t1, T2 t2, T3 t3) { }
}
```

当拦截器是泛型的时，原始包含类型和方法的类型参数从最外层到最内层作为类型参数传递给拦截器。在上述场景中，拦截器接收 `<int, bool, string>` 作为类型参数。如果拦截器类型参数的约束被这些类型参数违反，则会发生编译时错误。

这种替换允许拦截器使用在其声明处不在作用域内的类型参数。

```cs
using System.Runtime.CompilerServices;

class C
{
    public static void InterceptableMethod<T1>(T1 t) => throw null!;
}

static class Program
{
    public static void M<T2>(T2 t)
    {
        C.InterceptableMethod(t); // L1
    }
}

static class D
{
    [InterceptsLocation(1, "..(refers to call at L1)")]
    public static void Interceptor1<T2>(T2 t) => throw null!;
}
```

### 签名匹配

当调用被拦截时，拦截器和可拦截方法必须满足以下详述的签名匹配要求：
- 当将可拦截的实例方法与静态拦截器方法（包括经典扩展方法）进行比较时，我们将方法视为其扩展的简化形式进行比较。静态方法的第一个参数与实例方法的 `this` 参数进行比较。
    - 目前的实现要求拦截器必须是扩展方法才能进行此比较。我们计划在发布 .NET 8 之前解决此问题。
- 返回值和参数（包括 `this` 参数）必须具有相同的 ref 类型和类型。
- 如果在类型差异处发现类型对运行时不可区分，则报警告而非错误。例如，`object` 和 `dynamic`。
- 对于*安全*的可空性差异不报警告或错误，例如当可拦截方法接受 `string` 参数，而拦截器接受 `string?` 参数时。
- 方法名称和参数名称不要求匹配。
- 参数默认值不要求匹配。拦截时，拦截器方法上的默认值将被忽略。
- `params` 修饰符不要求匹配。
- `scoped` 修饰符和 `[UnscopedRef]` 必须等效。
- 通常影响调用站点行为的特性（例如 `[CallerLineNumber]`）在被拦截调用的拦截器上会被忽略。
  - 唯一的例外是当特性以影响安全性的方式影响方法的"能力"时，例如 `[UnscopedRef]`。此类特性要求在可拦截方法和拦截器方法之间匹配。

元数不需要在被拦截方法和拦截器方法之间匹配。换句话说，允许用非泛型拦截器拦截泛型方法。

### 冲突拦截器

如果多个拦截器引用同一位置，则为编译时错误。

如果编译中找到的 `[InterceptsLocation]` 特性未引用显式方法调用的位置，则为编译时错误。

### 拦截器可访问性

拦截器必须在发生拦截的位置可访问。

文件本地类型中包含的拦截器允许拦截另一文件中的调用，即使拦截器在调用站点处通常不*可见*。

这允许生成器作者避免用拦截器*污染查找*，有助于避免名称冲突，并防止从拦截器作者的角度在*非预期位置*使用拦截器。

我们也可能考虑调整 `[EditorBrowsable]` 的行为，使其在同一编译中生效。

### 结构体接收者捕获

其 `this` 参数按引用接受结构体的拦截器通常可以用来拦截结构体实例方法调用，前提是方法符合[签名匹配](#签名匹配)的要求。这包括即使在直接调用拦截器时不允许此类捕获的情况下，也必须隐式将接收者捕获到临时变量的情况。另请参阅标准中的 [12.8.9.3 扩展方法调用](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/expressions.md#12893-extension-method-invocations)。


```cs
using System.Runtime.CompilerServices;

struct S
{
    public void Original() { }
}

static class Program
{
    public static void Main()
    {
        new S().Original(); // L1: 拦截有效，无错误。
        new S().Interceptor(); // error CS1510: A ref or out value must be an assignable variable
    }
}

static class D
{
    [InterceptsLocation(1, "..(refers to call to 'Original()' at L1)")]
    public static void Interceptor(this ref S s)
}
```

我们允许对上述被拦截调用进行隐式接收者捕获的原因是：即使拦截器作者不拥有原始接收者类型，也需要能够进行拦截。如果不这样做，那么在上述示例中拦截 `Original()` 只能通过向 `struct S` 添加实例成员来实现。

### 编辑器体验

在此设计中，拦截器被视为编译后步骤。会对拦截器的误用给出诊断，但某些诊断只在命令行构建中给出，而不在 IDE 中给出。在编辑器中，对于编译中哪些调用实际上正在被拦截，可追溯性有限。

`GetInterceptorMethod(this SemanticModel, InvocationExpressionSyntax, CancellationToken)` 使分析器能够确定调用是否正在被拦截，如果是，则确定哪个方法正在拦截该调用。详见 https://github.com/dotnet/roslyn/issues/72093。

### 用户选择加入

要使用拦截器，用户项目必须指定属性 `<InterceptorsNamespaces>`。这是允许包含拦截器的命名空间列表。
```xml
<InterceptorsNamespaces>$(InterceptorsNamespaces);Microsoft.AspNetCore.Http.Generated;MyLibrary.Generated</InterceptorsNamespaces>
```

预计 `InterceptorsNamespaces` 列表中的每个条目大致对应一个源生成器。表现良好的组件不应将拦截器插入它们不拥有的命名空间。

为了兼容性，属性 `<InterceptorsPreviewNamespaces>` 可以用作 `<InterceptorsNamespaces>` 的别名。如果两个属性都有非空值，它们将按照 `$(InterceptorsNamespaces);$(InterceptorsPreviewNamespaces)` 的顺序连接后传递给编译器。

### 实现策略

在绑定阶段，`InterceptsLocationAttribute` 的用法被解码，每个用法的相关数据被收集在编译上的 `ConcurrentSet` 中：
- 被拦截的文件路径和位置
- 特性位置
- 被特性标记的方法符号

此时，会针对以下条件报告诊断：
- 特定于被标记拦截器方法本身的问题，例如它不是普通方法。
- 特定于引用位置的语法问题，例如它未引用[位置](#位置)小节中定义的适用简单名称。

在降级阶段，当给定的 `BoundCall` 被降级时：
- 我们检查其语法是否包含适用的简单名称
- 如果是，我们根据绑定阶段收集的关于 `InterceptsLocationAttribute` 的数据查找它是否正在被拦截。
- 如果它正在被拦截，我们在接收者和参数降级完成后执行额外步骤：
  - 在 `BoundCall` 上将可拦截方法替换为拦截器方法。
  - 如果拦截器是经典扩展方法，而可拦截方法是实例方法，我们调整 `BoundCall`，将接收者作为调用的第一个参数，将其他参数"向前推"，类似于如果原始调用使用简化形式的扩展方法进行绑定的方式。

此时，会针对以下条件报告诊断：
- 拦截器和可拦截方法之间的不兼容性，例如在其签名中。
- *重复*的 `[InterceptsLocation]`，即多个拦截器拦截同一调用。
- 拦截器在调用站点处不可访问。
