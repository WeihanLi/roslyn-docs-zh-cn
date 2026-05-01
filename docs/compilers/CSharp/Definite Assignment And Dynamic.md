# 动态表达式中的确定赋值与短路运算符

动态表达式中的确定赋值规则在之前的编译器中，允许某些可能导致读取未确定赋值变量的代码情况。详见 https://github.com/dotnet/roslyn/issues/4509 中的一个报告。

以下（复杂的）程序演示了"修复"这个问题会导致编译器允许读取未初始化变量的情况。

```cs
using System;

namespace ConsoleApp
{
    class Program
    {
        static void Main(string[] args)
        {
            string val = "unassigned";
            if (Api1() && Api2(out val) && UseVal(val))
            {
                Console.WriteLine(val);
            }
        }

        static dynamic Api1()
        {
            return new Waffle();
        }

        static bool Api2(out string val)
        {
            Console.WriteLine("Assigning to val");
            val = "assigned";
            return true;
        }

        static dynamic UseVal(string val)
        {
            Console.WriteLine($"Using val == {val}");
            return true;
        }
    }

    class Waffle
    {
        int count = 0;
        public static implicit operator bool(Waffle w)
        {
            try
            {
                return w.count != 0;
            }
            finally
            {
                w.count++;
            }
        }
    }
}
```

该程序的输出是：

```none
Using val == unassigned
unassigned
```

由于存在这种可能性，如果 `val` 没有初始值，编译器不得允许该程序编译。之前版本的编译器（VS2015 之前）即使 `val` 没有初始值也允许该程序编译。Roslyn 现在会诊断此尝试读取可能未初始化的变量。
