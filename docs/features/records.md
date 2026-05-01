C# 的记录
==============

记录是 C# 类和结构类型的一种新的、简化的声明形式，结合了许多更简单功能的优点。我们描述新功能（调用者-接收者参数和 *with 表达式*），给出记录声明的语法和语义，然后提供一些示例。

# 调用者-接收者参数

目前方法参数的*默认参数*必须是
- *常量表达式*；或
- 形如 `new S()` 的表达式，其中 `S` 是值类型；或
- 形如 `default(S)` 的表达式，其中 `S` 是值类型

我们扩展以添加以下内容
- 形如 `this.Identifier` 的表达式

这种新形式称为*调用者-接收者默认参数*，仅当满足以下所有条件时才允许
- 它出现的方法是实例方法；且
- 表达式 `this.Identifier` 绑定到封闭类型的实例成员，必须是字段或属性；且
- 它绑定的成员（以及 `get` 访问器，如果它是属性）至少与方法具有相同的可访问性；且
- `this.Identifier` 的类型通过标识或可空转换隐式转换为参数的类型（这是*默认参数*的现有约束）。

当从函数成员的调用中省略具有*调用者-接收者默认参数*的相应可选参数的参数时，接收者成员的值被隐式传递。

> **设计说明**：调用者-接收者参数的主要原因是支持 *with 表达式*。想法是您可以这样声明一个方法
> ```cs
> class Point
> {
>     public readonly int X;
>     public readonly int Y;
>     public Point With(int x = this.X, int y = this.Y) => new Point(x, y);
>     // 等
> }
> ```
> 然后这样使用它
> ```cs
>     Point p = new Point(3, 4);
>     p = p.With(x: 1);
> ```
> 创建一个与现有 `Point` 完全相同的新 `Point`，但更改了 `X` 的值。
> 
> 一旦我们支持调用者-接收者参数，是否值得添加 *with 表达式* 的语法形式是一个开放问题，因此我们可能会这样做*而不是*而不是*除了* *with 表达式*。

- [ ] **开放问题**：*调用者-接收者默认参数*相对于其他参数的评估顺序是什么？我们应该说它是未指定的吗？

# with 表达式

提出了一种新的表达式形式：

```antlr
primary_expression
    : with_expression
    ;

with_expression
    : primary_expression 'with' '{' with_initializer_list '}'
    ;

with_initializer_list
    : with_initializer
    | with_initializer ',' with_initializer_list
    ;

with_initializer
    : identifier '=' expression
    ;
```

标记 `with` 是一个新的上下文敏感关键字。

*with_initilaizer* 左侧的每个*标识符*必须绑定到 *with_expression* 的 *primary_expression* 类型的可访问实例字段或属性。给定 *with_expression* 的这些标识符之间可能没有重复的名称。

形如

> *e1* `with` `{` *identifier* = *e2*, ... `}`

的 *with_expression* 被视为形如

> *e1*`.With(`*identifier2*`:` *e2*, ...`)`

的调用

其中，对于每个名为 `With` 的方法，如果它是 *e1* 的可访问实例成员，我们选择 *identifier2* 作为该方法中第一个具有调用者-接收者参数的参数的名称，该参数是绑定到 *identifier* 的相同成员。如果无法识别这样的参数，则从考虑中消除该方法。要调用的方法是通过重载解析从剩余的候选者中选择的。

> **设计说明**：给定调用者-接收者参数，*with 表达式*的许多好处无需此特殊语法形式即可获得。因此，我们正在考虑是否需要它。它的主要好处是允许以字段和属性的名称而不是参数的名称进行编程。通过这种方式，我们提高了可读性和工具质量（例如，*with_expression* 标识符上的转到定义将导航到属性而不是方法参数）。

- [ ] **开放问题**：应修改此描述以支持扩展方法。
- [ ] **开放问题**：这种语法糖实际上值得吗？

# 模式匹配

有关 `operator is` 的规范及其与模式匹配的关系，请参阅[模式匹配规范](patterns.md)。

> **设计说明**：由于此处指定的编译器生成的 `operator is` 以及模式匹配的规范，记录声明
> ```cs
> public class Point(int X, int Y);
> ```
> 将支持位置模式匹配，如下所示
> ```cs
> Point p = new Point(3, 4);
> if (p is Point(3, var y)) { // 如果 X 是 3
>     Console.WriteLine(y);   // 打印 Y
> }
> ```

# 记录类型声明

`class` 或 `struct` 声明的语法扩展为支持值参数；参数成为类型的属性：

```antlr
class_declaration
    : attributes? class_modifiers? 'partial'? 'class' identifier type_parameter_list?
      record_parameters? record_class_base? type_parameter_constraints_clauses? class_body
    ;

struct_declaration
    : attributes? struct_modifiers? 'partial'? 'struct' identifier type_parameter_list?
      record_parameters? struct_interfaces? type_parameter_constraints_clauses? struct_body
    ;

record_class_base
    : class_type record_base_arguments?
    | interface_type_list
    | class_type record_base_arguments? ',' interface_type_list
    ;

record_base_arguments
    : '(' argument_list? ')'
    ;

record_parameters
    : '(' record_parameter_list? ')'
    ;

record_parameter_list
    : record_parameter
    | record_parameter record_parameter_list
    ;

record_parameter
    : attributes? type identifier record_property_name? default_argument?
    ;

record_property_name
    : ':' identifier
    ;

class_body
    : '{' class_member_declarations? '}'
    | ';'
    ;

struct_body
    : '{' struct_members_declarations? '}'
    | ';'
    ;
```

> **设计说明**：因为记录类型通常无需在 class-body 中显式声明任何成员即可使用，所以我们修改声明的语法以允许主体只是一个分号。

使用 *record-parameters* 声明的类（结构）称为*记录类*（*记录结构*），它们都是*记录类型*。

- [ ] **开放问题**：我们需要在语法中包含 *primary_constructor_body*，以便它可以出现在记录类型声明中。
- [ ] **开放问题**：参数名称的名称冲突规则是什么？大概不允许与类型参数或另一个 *record-parameter* 冲突。
- [ ] **开放问题**：我们需要指定 record-parameters 的范围。它们可以在哪里使用？大概至少可以在实例字段初始化器和 *primary_constructor_body* 中使用。
- [ ] **开放问题**：记录类型声明可以是部分的吗？如果是，参数必须在每个部分上重复吗？

### 记录类型的成员

除了在 *class-body* 中声明的成员外，记录类型还有以下额外成员：

#### 主构造函数

记录类型有一个 `public` 构造函数，其签名对应于类型声明的值参数。这称为类型的*主构造函数*，并导致隐式声明的*默认构造函数*被抑制。

在运行时，主构造函数

* 初始化与值参数对应的属性的编译器生成的支持字段（如果这些属性是编译器提供的；[参见 1.1.2](#1.1.2)）；然后
* 执行出现在 *class-body* 中的实例字段初始化器；然后
* 调用基类构造函数：
    * 如果 *record_base_arguments* 中有参数，则调用通过重载解析选择的具有这些参数的基构造函数；
    * 否则调用不带参数的基构造函数。
* 按源顺序执行每个 *primary_constructor_body*（如果有）的主体。

- [ ] **开放问题**：我们需要指定该顺序，特别是在部分编译单元之间。
- [ ] **开放问题**：我们需要指定每个显式声明的构造函数必须链接到主构造函数。
- [ ] **开放问题**：是否应该允许更改主构造函数的访问修饰符？
- [ ] **开放问题**：在记录结构中，没有记录参数是错误吗？

#### 主构造函数主体

```antlr
primary_constructor_body
    : attributes? constructor_modifiers? identifier block
    ;
```

*primary_constructor_body* 只能在记录类型声明中使用。*primary_constructor_body* 的*标识符*应命名声明它的记录类型。

*primary_constructor_body* 本身不声明成员，而是程序员为记录类型的主构造函数提供属性和指定访问权限的一种方式。它还使程序员能够提供在构造记录类型的实例时将执行的额外代码。

- [ ] **开放问题**：我们应该注意结构默认构造函数绑过了这个。
- [ ] **开放问题**：我们应该指定初始化的执行顺序。
- [ ] **开放问题**：我们应该允许在非记录类型声明中使用类似 *primary_constructor_body* 的东西（大概没有属性和修饰符），并像我们对待实例字段初始化器的代码一样对待它吗？

#### 属性

对于记录类型声明的每个记录参数，都有一个相应的 `public` 属性成员，其名称和类型取自值参数声明。其名称是 *record_property_name* 的*标识符*（如果存在），否则是 *record_parameter* 的*标识符*。如果没有显式声明或继承具有此名称和类型的具有 `get` 访问器的具体（即非抽象）公共属性，则编译器按如下方式生成它：

* 对于*记录结构*或 `sealed` *记录类*：
 * 生成一个 `private` `readonly` 字段作为 `readonly` 属性的支持字段。其值在构造期间使用相应主构造函数参数的值初始化。
 * 属性的 `get` 访问器实现为返回支持字段的值。
 * 覆盖每个"匹配"的继承虚拟属性的 `get` 访问器。

> **设计说明**：换句话说，如果您扩展一个基类或实现一个接口，该接口声明了与记录参数同名和同类型的公共抽象属性，则该属性被覆盖或实现。

- [ ] **开放问题**：当显式声明属性时，是否可以更改访问修饰符？
- [ ] **开放问题**：是否可以用字段替代属性？

#### Object 方法

对于*记录结构*或 `sealed` *记录类*，除非用户提供，否则编译器会生成方法 `object.GetHashCode()` 和 `object.Equals(object)` 的实现。

- [ ] **开放问题**：我们应该精确指定它们的实现。
- [ ] **开放问题**：我们还应该为记录类型添加接口 `IEquatable<T>` 并指定提供实现。
- [ ] **开放问题**：我们还应该指定我们实现每个 `IEquatable<T>.Equals`。
- [ ] **开放问题**：我们应该精确指定如何在面对记录继承时解决 Equals 问题：具体来说，我们如何生成相等方法使其是对称的、传递的、自反的等。
- [ ] **开放问题**：有人提议我们为记录类型实现 `operator ==` 和 `operator !=`。
- [ ] **开放问题**：我们应该自动生成 `object.ToString` 的实现吗？

#### `operator is`

记录类型有一个编译器生成的 `public static void operator is`，除非用户提供了具有任何签名的。其第一个参数是封闭记录类型，每个后续参数是与记录类型相应参数同名和同类型的 `out` 参数。此方法的编译器提供的实现应将每个 `out` 参数分配为相应属性的值。

有关 `operator is` 的语义，请参阅[模式匹配规范](patterns.md)。

#### `With` 方法

除非有用户声明的名为 `With` 的成员，否则记录类型有一个编译器提供的名为 `With` 的方法，其返回类型是记录类型本身，并包含与每个 *record-parameter* 相对应的一个值参数，这些参数以与记录类型声明中出现的相同顺序排列。每个参数应具有相应属性的*调用者-接收者默认参数*。

在 `abstract` 记录类中，编译器提供的 `With` 方法是抽象的。在记录结构或密封记录类中，编译器提供的 `With` 方法是 `sealed`。否则，编译器提供的 `With` 方法是 `virtual`，其实现应返回一个新实例，该实例通过以参数作为参数调用主构造函数从参数创建新实例，并返回该新实例。

- [ ] **开放问题**：我们还应该指定在什么条件下我们覆盖或实现继承的虚拟 `With` 方法或来自已实现接口的 `With` 方法。
- [ ] **开放问题**：我们应该说明当我们继承非虚拟 `With` 方法时会发生什么。

> **设计说明**：因为记录类型默认是不可变的，所以 `With` 方法提供了一种创建与现有实例相同但具有所选属性的新值的新实例的方法。例如，给定
> ```cs
> public class Point(int X, int Y);
> ```
> 有一个编译器提供的成员
> ```cs
>     public virtual Point With(int X = this.X, int Y = this.Y) => new Point(X, Y);
> ```
> 这使得记录类型的变量
> ```cs
> var p = new Point(3, 4);
> ```
> 可以被替换为具有一个或多个不同属性的实例
> ```cs
>     p = p.With(X: 5);
> ```
> 这也可以使用 *with_expression* 表达：
> ```cs
>     p = p with { X = 5 };
> ```

# 5. 示例

### 记录结构示例

此记录结构

```cs
public struct Pair(object First, object Second);
```

被翻译为此代码

```cs
public struct Pair : IEquatable<Pair>
{
    public object First { get; }
    public object Second { get; }
    public Pair(object First, object Second)
    {
        this.First = First;
        this.Second = Second;
    }
    public bool Equals(Pair other) // 用于 IEquatable<Pair>
    {
        return Equals(First, other.First) && Equals(Second, other.Second);
    }
    public override bool Equals(object other)
    {
        return (other as Pair)?.Equals(this) == true;
    }
    public override GetHashCode()
    {
        return (First?.GetHashCode()*17 + Second?.GetHashCode()).GetValueOrDefault();
    }
    public Pair With(object First = this.First, object Second = this.Second) => new Pair(First, Second);
    public static void operator is(Pair self, out object First, out object Second)
    {
        First = self.First;
        Second = self.Second;
    }
}
```

- [ ] **开放问题**：Equals(Pair other) 的实现应该是 Pair 的公共成员吗？
- [ ] **开放问题**：此 `Equals` 实现在面对继承时不是对称的。

> **设计说明**：因为一个记录类型可以从另一个继承，而此 `Equals` 实现在这种情况下不是对称的，所以它不正确。我们建议这样实现相等：
> ```cs
>     public bool Equals(Pair other) // 用于 IEquatable<Pair>
>     {
>         return other != null && EqualityContract == other.EqualityContract &&
>             Equals(First, other.First) && Equals(Second, other.Second);
>     }
>     protected virtual Type EqualityContract => typeof(Pair);
> ```
> 派生记录将 `override EqualityContract`。不太有吸引力的替代方案是限制继承。

### 密封记录示例

此密封记录类

```cs
public sealed class Student(string Name, decimal Gpa);
```

被翻译为此代码

```cs
public sealed class Student : IEquatable<Student>
{
    public string Name { get; }
    public decimal Gpa { get; }
    public Student(string Name, decimal Gpa)
    {
        this.Name = Name;
        this.Gpa = Gpa;
    }
    public bool Equals(Student other) // 用于 IEquatable<Student>
    {
        return other != null && Equals(Name, other.Name) && Equals(Gpa, other.Gpa);
    }
    public override bool Equals(object other)
    {
        return this.Equals(other as Student);
    }
    public override int GetHashCode()
    {
        return (Name?.GetHashCode()*17 + Gpa?.GetHashCode()).GetValueOrDefault();
    }
    public Student With(string Name = this.Name, decimal Gpa = this.Gpa) => new Student(Name, Gpa);
    public static void operator is(Student self, out string Name, out decimal Gpa)
    {
        Name = self.Name;
        Gpa = self.Gpa;
    }
}
```

### 抽象记录类示例

此抽象记录类

```cs
public abstract class Person(string Name);
```

被翻译为此代码

```cs
public abstract class Person : IEquatable<Person>
{
    public string Name { get; }
    public Person(string Name)
    {
        this.Name = Name;
    }
    public bool Equals(Person other)
    {
        return other != null && Equals(Name, other.Name);
    }
    public override Equals(object other)
    {
        return Equals(other as Person);
    }
    public override int GetHashCode()
    {
        return (Name?.GetHashCode()).GetValueOrDefault();
    }
    public abstract Person With(string Name = this.Name);
    public static void operator is(Person self, out string Name)
    {
        Name = self.Name;
    }
}
```

### 组合抽象和密封记录

给定上面的抽象记录类 `Person`，此密封记录类

```cs
public sealed class Student(string Name, decimal Gpa) : Person(Name);
```

被翻译为此代码

```cs
public sealed class Student : Person, IEquatable<Student>
{
    public override string Name { get; }
    public decimal Gpa { get; }
    public Student(string Name, decimal Gpa) : base(Name)
    {
        this.Name = Name;
        this.Gpa = Gpa;
    }
    public override bool Equals(Student other) // 用于 IEquatable<Student>
    {
        return Equals(Name, other.Name) && Equals(Gpa, other.Gpa);
    }
    public bool Equals(Person other) // 用于 IEquatable<Person>
    {
        return (other as Student)?.Equals(this) == true;
    }
    public override bool Equals(object other)
    {
        return (other as Student)?.Equals(this) == true;
    }
    public override int GetHashCode()
    {
        return (Name?.GetHashCode()*17 + Gpa.GetHashCode()).GetValueOrDefault();
    }
    public Student With(string Name = this.Name, decimal Gpa = this.Gpa) => new Student(Name, Gpa);
    public override Person With(string Name = this.Name) => new Student(Name, Gpa);
    public static void operator is(Student self, out string Name, out decimal Gpa)
    {
        Name = self.Name;
        Gpa = self.Gpa;
    }
}
```
