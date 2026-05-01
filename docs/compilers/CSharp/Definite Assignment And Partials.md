# 分部方法和条件方法的确定赋值

C# 语言规范中关于调用表达式确定赋值的部分 [5.3.2.2] 错误地假设调用表达式及其参数在到达时将在运行时被求值。对于*分部方法* [10.2.7] 和标记了*条件属性* [17.4.2] 的方法，这可能并不正确。

C# 编译器正确考虑了方法调用（和参数求值）可能被省略的情况。

```cs
using System.Diagnostics;

 static partial class A
{
    static void Main()
    {
        int x;
        x.Foo(); // 正常
        Bar(x); // 正常

        Bar(x = 1);
        x.ToString(); // error CS0165: 使用了未赋值的局部变量 'x'
    }

    static partial void Foo(this int y);

    [Conditional("xxxxxx")]
    static void Bar(int y) { }

}
```
