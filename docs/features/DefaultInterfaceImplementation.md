默认接口实现
=========================

*默认接口实现*功能允许在接口声明中为接口成员提供默认实现。

以下是提案链接：https://github.com/dotnet/csharplang/blob/main/proposals/csharp-8.0/default-interface-methods.md

**支持的内容：**
- 在普通接口方法的声明中提供实现，并在类型实现该接口时将其识别为该方法的默认实现。
示例如下：
```
public interface I1
{
    void M1() 
    {
        System.Console.WriteLine("Default implementation of M1 is called!!!");
    }
}

class Test1 : I1
{
    static void Main()
    {
        I1 x = new Test1();
        x.M1();
    }
}
```

- 在派生接口中重新抽象化接口实现。即使基接口提供了实现，实现派生接口的类型也必须提供实现。
示例如下：
```
public interface I1
{
    void M1() 
    {
    }

    int P1 
    {
        get => throw null;
        set => throw null;
    }

    event System.Action E1
    {
        add => throw null;
        remove => throw null;
    }
}

public interface I2 : I1
{
    abstract void I1.M1();
    abstract int I1.P1 {get;set;}
    abstract event System.Action I1.E1;
}

class Test1 : I2 // 该类型必须实现 I1 的所有成员
{
}
```
在元数据中，表示重新抽象化的方法同时具有 `abstract`、`virtual`、`sealed` 三个标志，且不具有 `newslot` 标志。


- 在属性或索引器的声明中提供实现，并在类型实现该接口时将其识别为属性或索引器的默认实现。

- 在事件的声明中提供实现，并在类型实现该接口时将其识别为事件的默认实现。

- 在接口方法中使用 **partial**、**public**、**internal**、**private**、**protected**、**static**、**virtual**、**sealed**、**abstract**、**extern** 和 **async** 修饰符。

- 在接口属性中使用 **public**、**internal**、**private**、**protected**、**static**、**virtual**、**sealed**、**abstract** 和 **extern** 修饰符。

- 在接口索引器中使用 **public**、**internal**、**private**、**protected**、**virtual**、**sealed**、**abstract** 和 **extern** 修饰符。

- 在接口属性/索引器访问器中使用 **internal**、**private**、**protected** 修饰符。

- 在接口事件中使用 **public**、**internal**、**private**、**protected**、**static**、**virtual**、**sealed**、**abstract** 和 **extern** 修饰符。

- 在接口中声明类型。

- 在派生接口中使用显式实现语法实现接口方法，可访问性为 **protected**，允许的修饰符：**extern** 和 **async**。

- 在派生接口中使用显式实现语法实现接口属性和索引器，可访问性为 **protected**，允许的修饰符：**extern**。

- 在派生接口中使用显式实现语法实现接口事件，可访问性为 **protected**，不允许其他修饰符。

- 声明静态字段、自动属性和类似字段的事件。

- 在接口中声明运算符 ```+ - ! ~ ++ -- true false * / % & | ^ << >> > < >= <=```。

- 基访问
以下形式的基访问已添加（https://github.com/dotnet/csharplang/blob/main/meetings/2018/LDM-2018-11-14.md）
```
    base ( <type-syntax> )  .   identifier
    base ( <type-syntax> )   [   argument-list   ]
```

`type-syntax` 可引用包含类型的某个基类，或包含类型实现或继承的某个接口。

当 `type-syntax` 引用某个类时，成员查找规则、重载解析规则和 IL 生成规则与 7.3 支持的基访问形式的规则一致，区别在于使用指定的基类而非直接基类，且找到的最派生实现必须是该类的成员。

当 `type-syntax` 引用某个接口时：
1. 在该接口中执行成员查找，使用接口内的常规成员查找规则，但 System.Object 的成员不参与查找。
2. 对查找过程返回的成员执行常规重载解析，虚方法或抽象方法在此步骤中不会被最派生实现替换（与 `type-syntax` 引用类时不同）。若重载解析结果为虚方法或抽象方法，则该方法必须在指定接口类型中具有实现，否则报错。该实现必须在调用位置可访问。若重载解析结果为非虚方法，则该方法必须在指定接口类型中声明。
3. 在 IL 生成期间，使用 **call**（非虚调用）指令来调用方法。若上一步重载解析的结果为虚方法或抽象方法，则使用指定接口中该方法的实现作为指令的目标。
   
考虑到最具体接口实现的可访问性要求，在派生接口中提供的实现的可访问性更改为 **protected**。

**未解决问题和工作项**在 https://github.com/dotnet/roslyn/issues/17952 中跟踪。

**ECMA-335 中将变得过时/不准确/不完整的部分**
>I.8.5.3.2 成员和嵌套类型的可访问性
接口定义的成员（嵌套类型除外）应为 public。
I.8.9.4 接口类型定义
类似地，接口类型定义不应为其类型的值提供任何方法的实现。
接口可以有静态或虚方法，但不得有实例方法。
但是，由于可访问性特性是相对于实现类型而非接口本身的，接口的所有成员应具有 public 可访问性，……
I.8.11.1 方法定义
接口定义的所有非静态方法均为抽象方法。
接口定义中的所有非静态方法定义均为虚方法。
II.10.4 方法实现要求
II.12 接口语义
接口可以有静态字段和方法，但不得有实例字段或方法。接口可以定义虚方法，但这些方法必须是抽象方法（参见 Partition I 和 §II.15.4.2.4）。
II.12.2 在接口上实现虚方法
如果类定义了任何名称和签名与接口上虚方法匹配的 public 虚方法，则按类型声明顺序将其添加到该方法的列表中。
如果该类上（直接或继承）存在与接口方法名称和签名相同的 public 虚方法，且其泛型类型参数与该类或其继承链中任何类的现有列表中的方法不完全匹配，则按类型声明顺序将其添加到接口上对应方法的列表中。
II.15.2 静态、实例和虚方法
因此，实例方法只能在类或值类型中定义，不能在接口中或类型外部（即全局范围）定义。
II.22.27 MethodImpl：0x19
MethodBody 索引的方法应为 Class 或 Class 的某个基类的成员（MethodImpl 不允许编译器"挂接"任意方法体）
II.22.37 TypeDef：0x02
接口拥有的所有方法（Flags.Interface = 1）均应为抽象方法（Flags.Abstract = 1）
IV.6 系统库的实现特定修改
不应向现有接口添加接口和虚方法。
