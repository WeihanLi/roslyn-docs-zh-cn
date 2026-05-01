元组快速入门指南（C# 7.0 和 Visual Basic 15）
------------------------------------
1. 安装 VS2017
2. 启动一个 C# 或 VB 项目
3. 从 NuGet 添加对 `System.ValueTuple` 包的引用
![安装 ValueTuple 包](img/install-valuetuple.png)
4. 在 C# 中使用元组：

```C#
public class C
{
        public static (int code, string message) Method((int, string) x) 
        { 
                return x;
        }

        public static void Main()
        {
                var pair1 = (42, "hello");
                System.Console.Write(Method(pair1).message);
        
                var pair2 = (code: 43, message: "world");
                System.Console.Write(pair2.message);
        }
}
```

5. 或在 VB 中使用元组：

```VB
Public Class C
        Public Shared Function Method(x As (Integer, String)) As (code As Integer, message As String)
                Return x
        End Function

        Public Shared Sub Main()
                Dim x = (42, "hello")
                System.Console.Write(C.Method(x).message)
        
                Dim pair2 = (code:=43, message:="world")
                System.Console.Write(pair2.message)
        End Sub
End Class
```

6. 使用解构（仅限 C#）：请参阅[解构页面](deconstruction.md)

如果没有来自 NuGet 的 `System.ValueTuple` 包，编译器将产生错误：
``error CS8179: Predefined type 'System.ValueTuple`2' is not defined or imported``

设计
------
本文档的目标是记录正在实施的内容。随着设计的演变，文档将进行调整。
设计中的更改将在实施更改时应用于本文档。

元组类型
-----------

元组类型将使用与参数列表非常相似的语法引入：

```C#
public (int sum, int count) Tally(IEnumerable<int> values) { ... }
    
var t = Tally(myValues);
Console.WriteLine($"Sum: {t.sum}, count: {t.count}");   
 ```

语法 `(int sum, int count)` 表示具有给定名称和类型的公共字段的匿名数据结构，也称为*元组*。

在 C# 中无需进一步添加语法，元组值可以创建为
```C#
var t1 = new (int sum, int count) (0, 1);
var t2 = new (int sum, int count) { sum = 0, count = 0 };
var t3 = new (int, int) (0, 1);     // 字段名称是可选的    
```
请注意，指定字段名称是可选的；但是，当提供名称时，所有字段都必须命名。不允许重复名称。

元组字面量
--------------

```C#
var t1 = (0, 2);				// 从值推断元组类型
var t2 = (sum: 0, count: 1);	// 从名称和值推断元组类型
```

创建已知目标类型的元组值：
```C#
public (int sum, int count) Tally(IEnumerable<int> values) 
{
    var s = 0; var c = 0;
    foreach (var value in values) { s += value; c++; }
    return (s, c); // 目标类型为 (int sum, int count)
}
```

请注意，指定字段名称是可选的，但当提供名称时，所有字段都必须命名。不允许重复名称。

```C#
var t1 = (sum: 0, 1);		// 错误！某些字段有名称，某些没有。
var t2 = (sum: 0, sum: 1);	// 错误！重复名称。
```

与底层类型的对偶性
--------------

元组映射到特定名称的底层类型 - 

```
System.ValueTuple<T1, T2>
System.ValueTuple<T1, T2, T3>
...
System.ValueTuple<T1, T2, T3,..., T7, TRest>
```

在所有场景中，元组类型的行为与底层类型完全相同，只是程序员给出的更具表现力的字段名称是额外的可选增强。

```C#
var t = (sum: 0, count: 1);	
t.sum   = 1;				// sum   是字段 #1 的名称 
t.Item1 = 1;				// Item1 是底层字段 #1 的名称，也可用

var t1 = (0, 1);			// 元组省略字段名称。
t.Item1 = 1;				// 底层字段名称仍然可用 

t.ToString()				// 调用底层元组类型上的 ToString。

System.ValueTuple<int, int> vt = t;	// 恒等转换 
(int moo, int boo) t2 = vt;			// 恒等转换
```

由于元组的双重性质，不允许分配与底层类型的预先存在的成员名称重叠的字段名称。
唯一的例外是在相应位置 N 使用预定义的"Item1"、"Item2"、..."ItemN"，因为这不会产生歧义。

```C#
var t =  (ToString: 0, ObjectEquals: 1);		// 错误：名称与底层成员名称匹配
var t1 = (Item1: 0, Item2: 1);					// 有效
var t2 = (misc: 0, Item1: 1);					// 错误："Item1"在错误的位置使用
```

底层元组类型实现的示例：
https://github.com/dotnet/roslyn/blob/features/tuples/docs/features/ValueTuples.cs
将由 FX Core 中的实际元组实现替换。

恒等转换
--------------

元素名称与元组转换无关。具有相同顺序的相同类型的元组可以彼此进行恒等转换，或者与相应的底层 ValueTuple 类型进行恒等转换，无论名称如何。

```C#
var t = (sum: 0, count: 1);	

System.ValueTuple<int, int> vt = t;  // 恒等转换 
(int moo, int boo) t2 = vt;  		 // 恒等转换

t2.moo = 1;
```
也就是说，如果您在转换的一侧的*一个*位置有一个元素名称，而在另一侧的*另一个*位置有相同的名称。这将表明代码几乎肯定包含错误：

```C#
(int sum, int count) moo ()
{
	return (count: 1, sum: 3); // 警告!! 
}	
```

重载、覆盖、隐藏
--------------
出于重载覆盖隐藏的目的，相同类型和长度的元组以及它们的底层 ValueTuple 类型被认为是等效的。所有其他差异都不重要。

覆盖成员时，允许使用与基成员中相同或不同的字段名称的元组类型。

在基成员和派生成员签名之间的不匹配字段使用相同字段名称的情况下，编译器会报告警告。

```C#
class Base 
{
	virtual void M1(ValueTuple<int, int> arg){...}
}
class Derived : Base
{
	override void M1((int c, int d) arg){...} // 有效覆盖，签名等效
}
class Derived2 : Derived 
{
	override void M1((int c1, int c) arg){...} // 也有效，对名称 'c' 可能的误用发出警告 
}

class InvalidOverloading 
{
	virtual void M1((int c, int d) arg){...}
	virtual void M1((int x, int y) arg){...}			// 无效重载，签名等效
	virtual void M1(ValueTuple<int, int> arg){...}		// 也无效
}
```

运行时的名称擦除
--------------
重要的是，元组字段名称不是元组运行时表示的一部分，而是仅由编译器跟踪。

因此，字段名称对元组实例的第三方观察者不可用 - 例如反射或动态代码。

与恒等转换一致，装箱的元组不保留字段名称，并将解箱为具有相同顺序的相同元素类型的任何元组类型。

```C#
object o = (a: 1, b: 2);    		 // 装箱转换 
var t = ((int moo, int boo))o;		 // 解箱转换
```

目标类型化
--------------

元组字面量在指定元组类型的上下文中使用时是"目标类型化"的。这意味着元组字面量具有"从表达式转换"到任何元组类型，只要元组字面量的元素表达式对元组类型的元素类型具有隐式转换。

```C#
(string name, byte age) t = (null, 5); // Ok：表达式 null 和 5 转换为 string 和 byte
```

在元组字面量不是转换的一部分的情况下，元组使用其"自然类型"，这意味着元素类型是组成表达式类型的元组类型。由于不是所有表达式都有类型，所以不是所有元组字面量都有自然类型：

```C#
var t = ("John", 5);  						//   Ok：t 的类型是 (string, int)
var t = (null, 5);    						//   错误：元组表达式没有类型，因为 null 没有类型
((1,2, null), 5).ToString();    	    	//   错误：元组表达式没有类型

ImmutableArray.Create((()=>1, 1));        	//   错误：元组表达式没有类型，因为 lambda 没有类型
ImmutableArray.Create(((Func<int>)(()=>1), 1)); //   ok
```

元组字面量可以包含名称，在这种情况下它们成为自然类型的一部分：

```C#
var t = (name: "John", age: 5); // t 的类型是 (string name, int age)
t.age++;						// t 有一个名为 age 的字段。
```

从元组表达式到元组类型的成功转换被归类为 _ImplictTuple_ 转换，除非元组的自然类型与目标类型完全匹配，在这种情况下它是 _Identity_ 转换。

```C#
void M1((int x, int y) arg){...};
void M1((object x, object y) arg){...};

M1((1, 2));   			// 使用第一个重载。恒等转换优于隐式转换。 
M1(("hi", "hello"));   // 使用第二个重载。隐式元组转换优于无转换。 
```

目标类型化将"透视"可空目标类型。从元组表达式到可空元组类型的成功转换被归类为 _ImplicitNullable_ 转换。

```C#
((int x, int y, int z)?, int t)? SpaceTime()
{
	return ((1,2,3), 7); 	// 有效，隐式可空转换
}
``` 

重载解析和没有自然类型的元组
--------------

在重载解析期间可能会出现这样的情况：由于没有自然类型的元组参数可以隐式转换为相应的参数，多个同样合格的候选者可能可用。

在这种情况下，重载解析采用 _精确匹配_ 规则，其中没有自然类型的参数根据组成结构元素递归匹配相应参数的类型。

元组表达式也不例外，_精确匹配_ 规则基于组成元组参数的自然类型。

该规则相对于其他不具有自然类型的包含或被包含表达式是相互递归的。

```C#
void M1((int x, Func<(int, int)>) arg){...};
void M1((int x, Func<(int, byte)>) arg){...};

M1((1, ()=>(2, 3)));         // 由于"精确匹配"规则，使用第一个重载
``` 

转换
--------------

元组类型和表达式通过将元素的转换"提升"为整体 _元组转换_ 来支持各种转换。
出于分类目的，所有元素转换都被递归考虑。例如：要进行隐式转换，所有元素表达式/类型必须对相应的元素类型具有隐式转换。

元组转换是*标准转换*，因此可以与用户定义的运算符堆叠以形成用户定义的转换。

只要所有元素转换都可用作实例转换，元组转换就可以被归类为扩展方法调用的有效实例转换。

语言语法更改
---------------------
这基于 Lucian 的 [ANTLR 语法](https://raw.githubusercontent.com/ljw1004/csharpspec/gh-pages/csharp.g4)。

对于元组类型声明：

```ANTLR
struct_type
    : type_name
    | simple_type
    | nullable_type
    | tuple_type // 新增
    ; 
    
tuple_type
    : '(' tuple_type_element_list ')'
    ;
    
tuple_type_element_list
    : tuple_type_element ',' tuple_type_element
    | tuple_type_element_list ',' tuple_type_element
    ;
    
tuple_type_element
    : type identifier?
    ;
```

对于元组字面量：

```ANTLR
literal
    : boolean_literal
    | integer_literal
    | real_literal
    | character_literal
    | string_literal
    | null_literal
    | tuple_literal // 新增
    ;

tuple_literal
    : '(' tuple_literal_element_list ')'
    ;

tuple_literal_element_list
    : tuple_literal_element ',' tuple_literal_element
    | tuple_literal_element_list ',' tuple_literal_element
    ;

tuple_literal_element
    : ( identifier ':' )? expression
    ;
```

请注意，由于它不是常量表达式，元组字面量不能用作可选参数的默认值。

待解决问题：
-----------

- [ ] 提供有关元组类型声明的语义的更多详细信息，包括静态（类型规则、约束、全有或全无名称、不能用于 'is' 的右侧，...）和动态（运行时做什么？）。
- [ ] 提供有关元组字面量的语义的更多详细信息，包括静态（从表达式的新类型转换、从类型的新类型转换、全有或全无、混乱的名称、底层类型、底层名称、列出此类型的成员、访问意味着什么，）和动态（执行此转换时会发生什么？）。
- [ ] 精确匹配表达式

参考：
-----------

功能的许多细节和动机在
[提案：元组的语言支持](https://github.com/dotnet/roslyn/issues/347)中给出

[2016年4月6日的 C# 设计笔记](https://github.com/dotnet/roslyn/issues/10429)
