# 接口多重继承查找的歧义

旧版本的本机编译器和 Roslyn 编译器在处理接口中多重继承的名称查找歧义时，都不遵守 C# 语言规范。两种编译器都接受以下代码（带警告），尽管 C# 语言规范第 7.4 节（最后一个项目符号）要求发生错误。

```cs
delegate void D();
interface I1
{
    void M();
}

interface I2
{
    event D M;
}

interface I3 : I1, I2 { }
public class P : I3
{
    event D I2.M
    {
        add { }
        remove { }
    }

    void I1.M()
    {
    }
}

class Q : P
{
    static int Main(string[] args)
    {
        Q p = new Q();
        I3 m = p;
        m.M(); // C# 规范 7.4（7.4.1 前的最后一个项目符号）要求绑定时错误。编译器仅给出警告。
        return 0;
    }
}
```

编译器中实现的规则是：当沿不同继承路径找到方法和非方法时，编译器发出警告并丢弃非方法。
