# SkipLocalsInit 功能设计

该功能向编译器添加了一个新的已知特性 `System.Runtime.CompilerServices.SkipLocalsInit`，它使嵌套在该特性范围"内部"的方法体省略 `.locals init` CIL 指令——该指令会导致 CLR 对本地变量以及通过 `localloc` 指令保留的空间进行零初始化。"嵌套在特性范围内部"的含义如下定义：

1. 当该特性应用于模块时，该模块中所有已生成的方法（包括生成的方法）都将跳过本地初始化。

2. 当该特性应用于类型（包括接口）时，该类型内部所有传递性方法（包括生成的方法）都将跳过本地初始化。这也包括嵌套类型。

3. 当该特性应用于方法时，该方法以及编译器生成的所有包含该方法内部用户影响代码的方法（例如本地函数、lambda 表达式、async 方法）都将跳过本地初始化。

对于以上情形，仅当生成的方法内部有用户代码，或有大量编译器生成的局部变量时，才会跳过本地初始化。例如，async 方法的 MoveNext 方法将跳过本地初始化，但显示类的构造函数可能不会，因为显示类构造函数通常根本没有局部变量。

一些值得注意的决策：

1. 虽然 SkipLocalsInit 不要求使用 unsafe 代码块，但它确实需要向编译器传递 `unsafe` 标志。这是为了表明在某些情况下代码可能读取未初始化的内存，包括 `stackalloc`。

2. 由于在生成 netmodule 时编译器无法控制其他模块的 localsinit，即使 SkipLocalsInitAttribute 的 AttributeUsage 指定了允许在程序集级别使用，SkipLocalsInit 在程序集级别也不受支持。

3. SkipLocalsInitAttribute 的 `Inherited` 属性不受支持。SkipLocalsInit 不会通过继承传播。
