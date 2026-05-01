# System.TypedReference

[此为占位内容。我们需要在此处补充更多文档]

这是一封旧电子邮件对话，提供了 C# 编译器中互操作支持的一些背景信息。讽刺的是，这段对话表明我们永远不会记录它！

-------------------
主题：RE: error CS0610: 字段或属性不能是 'System.TypedReference' 类型
发件人：Eric Lippert
收件人：Aleksey Tsingauz; Neal Gafter; Roslyn Compiler Dev Team
发送时间：2011 年 1 月 24 日（星期一）上午 9:42

基本上发生的事情是，我们有一些未记录的功能，让你可以在不知道变量编译时类型的情况下传递对**变量**的引用。我们拥有这个功能的原因是为了在 CLR 中启用 C 风格的"varargs"；你可能有一个接受未指定数量参数的方法，其中一些参数可能是对变量的引用。

因为 `TypedReference` 可以包含栈分配变量的地址，你不允许将它们存储在字段中，就像你不允许创建 ref 类型的字段一样。这样我们就知道我们永远不会存储对"死亡"栈变量的引用。

我们在本机编译器中有一堆代码，确保类型引用（以及其他一些同样神奇的类型）不被错误使用；我们将来某个时候必须在 Roslyn 中做同样的事情。

这些东西都没有很好的文档记录。我们有四个未记录的语言关键字，让你可以操作类型引用；据我所知，我们无意记录它们。它们只存在于 C# 代码需要与 C 风格方法互操作的少数情况。

关于这些功能的一些文章，如果你有兴趣可能会提供更多背景信息：

http://www.eggheadcafe.com/articles/20030114.asp

http://stackoverflow.com/questions/4764573/why-is-typedreference-behind-the-scenes-its-so-fast-and-safe-almost-magical

http://stackoverflow.com/questions/1711393/practical-uses-of-typedreference

http://stackoverflow.com/questions/2064509/c-type-parameters-specification

http://stackoverflow.com/questions/4046397/generic-variadic-parameters

http://bartdesmet.net/blogs/bart/archive/2006/09/28/4473.aspx

此致，
Eric

---------------------
发件人：Aleksey Tsingauz 
发送时间：2011 年 1 月 23 日（星期日）晚上 11:00
收件人：Neal Gafter; Roslyn Compiler Dev Team
主题：RE: error CS0610: 字段或属性不能是 'System.TypedReference' 类型
 
我认为这是关于 ECMA-335 §8.2.1.1 托管指针和相关类型：
 
**托管指针**（§12.1.1.2）或 **byref**（§8.6.1.3、§12.4.1.5.2）可以指向局部变量、参数、复合类型的字段或数组的元素。但是，当调用跨越远程边界时（见 §12.5），符合规范的实现可以使用复制进/复制出机制而不是托管指针。因此，程序不应依赖真实指针的别名行为。托管指针类型只允许用于局部变量（§8.6.1.3）和参数签名（§8.6.1.4）；不能用于字段签名（§8.6.1.2）、作为数组的元素类型（§8.9.1），装箱托管指针类型的值是不允许的（§8.2.4）。将托管指针类型用作方法的返回类型（§8.6.1.5）是不可验证的（§8.8）。

基类库（见 Partition IV Library）中有三个特殊值类型：`System.TypedReference`、`System.RuntimeArgumentHandle` 和 `System.ArgIterator`；CLI 对它们进行特殊处理。
值类型 `System.TypedReference`，或*类型引用*或 *typedref*，（§8.2.2、§8.6.1.3、§12.4.1.5.3）同时包含指向位置的托管指针和可以存储在该位置的类型的运行时表示。类型引用具有与 byref 相同的限制。类型引用由 CIL 指令 `mkrefany`（见 Partition III）创建。

值类型 `System.RuntimeArgumentHandle` 和 `System.ArgIterator`（见 Partition IV 和 Partition III 中的 CIL 指令 `arglist`）包含指向 VES 栈的指针。它们可以用于局部变量和参数签名。将这些类型用于字段、方法返回类型、数组的元素类型或装箱中是不可验证的（§8.8）。这两种类型称为 *byref-like* 类型。

----------------
发件人：Neal Gafter 
发送时间：2011 年 1 月 23 日（星期日）晚上 8:37
收件人：Roslyn Compiler Dev Team
抄送：Neal Gafter
主题：error CS0610: 字段或属性不能是 'System.TypedReference' 类型
 
这个错误是什么意思？在哪里有文档记录？
