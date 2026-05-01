`private protected` 访问修饰符
=================================

我们提议添加一种新的访问修饰符组合 `private protected`（可以以任意顺序出现在修饰符中）。这对应于 CLR 中 protectedAndInternal 的概念，并借用了 [C++/CLI](https://msdn.microsoft.com/en-us/library/ke3a209d.aspx#BKMK_Member_visibility) 中目前使用的相同语法。

声明为 `private protected` 的成员可以在其容器的子类中访问，但前提是该子类与该成员处于同一程序集中。

我们对语言规范进行如下修改（新增内容以粗体显示）。下面不显示章节编号，因为它们可能因规范版本而异。

-----

> 成员声明的可访问性可以是以下之一：
- 公共（Public），通过在成员声明中包含 public 修饰符来选择。public 的直观含义是"访问不受限制"。
- 受保护（Protected），通过在成员声明中包含 protected 修饰符来选择。protected 的直观含义是"访问仅限于包含类或从包含类派生的类型"。
- 内部（Internal），通过在成员声明中包含 internal 修饰符来选择。internal 的直观含义是"访问仅限于此程序集"。
- 受保护内部（Protected internal），通过在成员声明中同时包含 protected 和 internal 修饰符来选择。protected internal 的直观含义是"在此程序集内以及从包含类派生的类型中可访问"。
- **私有受保护（Private protected），通过在成员声明中同时包含 private 和 protected 修饰符来选择。private protected 的直观含义是"在此程序集内由从包含类派生的类型可访问"。**

-----

> 根据成员声明所处的上下文，只允许某些类型的声明可访问性。此外，当成员声明不包含任何访问修饰符时，声明所在的上下文决定了默认声明可访问性。
- 命名空间隐式具有 public 声明可访问性。命名空间声明上不允许使用访问修饰符。
- 直接在编译单元或命名空间中声明的类型（而不是在其他类型中）可以具有 public 或 internal 声明可访问性，默认为 internal 声明可访问性。
- 类成员可以具有五种声明可访问性中的任意一种，默认为 private 声明可访问性。[注意：声明为类成员的类型可以具有五种声明可访问性中的任意一种，而声明为命名空间成员的类型只能具有 public 或 internal 声明可访问性。结束注意]
- 结构体成员可以具有 public、internal 或 private 声明可访问性，默认为 private 声明可访问性，因为结构体是隐式密封的。结构体中引入的结构体成员（即非继承自该结构体的成员）不能具有 protected **、** ~~或~~ protected internal **或 private protected** 声明可访问性。[注意：声明为结构体成员的类型可以具有 public、internal 或 private 声明可访问性，而声明为命名空间成员的类型只能具有 public 或 internal 声明可访问性。结束注意]
- 接口成员隐式具有 public 声明可访问性。接口成员声明上不允许使用访问修饰符。
- 枚举成员隐式具有 public 声明可访问性。枚举成员声明上不允许使用访问修饰符。

-----

> 在程序 P 中类型 T 内声明的嵌套成员 M 的可访问性域定义如下（注意 M 本身可能是一个类型）：
- 如果 M 的声明可访问性为 public，则 M 的可访问性域就是 T 的可访问性域。
- 如果 M 的声明可访问性为 protected internal，设 D 为 P 的程序文本与在 P 外部声明的任何从 T 派生的类型的程序文本的并集。M 的可访问性域为 T 的可访问性域与 D 的交集。
- **如果 M 的声明可访问性为 private protected，设 D 为 P 的程序文本与在 P 外部声明的任何从 T 派生的类型的程序文本的交集。M 的可访问性域为 T 的可访问性域与 D 的交集。**
- 如果 M 的声明可访问性为 protected，设 D 为 T 的程序文本与任何从 T 派生的类型的程序文本的并集。M 的可访问性域为 T 的可访问性域与 D 的交集。
- 如果 M 的声明可访问性为 internal，则 M 的可访问性域为 T 的可访问性域与 P 的程序文本的交集。
- 如果 M 的声明可访问性为 private，则 M 的可访问性域为 T 的程序文本。

-----

> 当受保护**或 private protected** 实例成员在其声明所在类的程序文本之外被访问，以及当受保护内部实例成员在其声明所在程序的程序文本之外被访问时，访问必须在派生自其声明类的类声明中进行。此外，访问还必须通过该派生类类型或由其构造的类类型的实例进行。此限制防止一个派生类访问其他派生类的受保护成员，即使这些成员继承自同一个基类。

-----

> 类型声明允许的访问修饰符和默认访问取决于声明发生的上下文（§9.5.2）：
- 在编译单元或命名空间中声明的类型可以具有 public 或 internal 访问。默认为 internal 访问。
- 在类中声明的类型可以具有 public、protected internal、**private protected**、protected、internal 或 private 访问。默认为 private 访问。
- 在结构体中声明的类型可以具有 public、internal 或 private 访问。默认为 private 访问。

-----

> 静态类声明受以下限制约束：
- 静态类不得包含 sealed 或 abstract 修饰符。（但是，由于静态类不能被实例化或派生，它的行为就好像它既是 sealed 也是 abstract 的。）
- 静态类不得包含类基规范（§16.2.5），不能显式指定基类或已实现接口的列表。静态类隐式继承自 object 类型。
- 静态类只能包含静态成员（§16.4.8）。[注意：所有常量和嵌套类型都归类为静态成员。结束注意]
- 静态类不得具有带有 protected **、private protected、**或 protected internal 声明可访问性的成员。

> 违反这些限制中的任何一条都是编译时错误。

-----

> 类成员声明可以具有~~五~~**六**种可能的声明可访问性（§9.5.2）中的任意一种：public、**private protected**、protected internal、protected、internal 或 private。除 protected internal **和 private protected** 组合之外，指定多个访问修饰符是编译时错误。当类成员声明不包含任何访问修饰符时，假定为 private。

-----

> 非嵌套类型可以具有 public 或 internal 声明可访问性，默认具有 internal 声明可访问性。嵌套类型也可以具有这些形式的声明可访问性，以及一种或多种其他形式的声明可访问性，具体取决于包含类型是类还是结构体：
- 在类中声明的嵌套类型可以具有~~五~~**六**种形式的声明可访问性中的任意一种（public、**private protected**、protected internal、protected、internal 或 private），并且与其他类成员一样，默认为 private 声明可访问性。
- 在结构体中声明的嵌套类型可以具有三种形式的声明可访问性中的任意一种（public、internal 或 private），并且与其他结构体成员一样，默认为 private 声明可访问性。

-----

> 被 override 声明覆盖的方法称为被覆盖的基方法。对于在类 C 中声明的 override 方法 M，通过检查 C 的每个基类类型（从 C 的直接基类类型开始并继续到每个连续的直接基类类型）来确定被覆盖的基方法，直到在给定的基类类型中找到至少一个在替换类型参数后具有与 M 相同签名的可访问方法为止。为了定位被覆盖的基方法，如果某方法是 public 的、protected 的、protected internal 的，或者是 **internal 或 private protected** 并在与 C 相同的程序中声明的，则认为该方法是可访问的。

-----

> 访问器修饰符的使用受以下限制约束：
- 访问器修饰符不得在接口或显式接口成员实现中使用。
- 对于没有 override 修饰符的属性或索引器，只有当属性或索引器同时具有 get 和 set 访问器时才允许使用访问器修饰符，且只允许在其中一个访问器上使用。
- 对于包含 override 修饰符的属性或索引器，访问器必须匹配被覆盖访问器的访问器修饰符（如果有）。
- 访问器修饰符声明的可访问性必须比属性或索引器本身声明的可访问性严格受限。具体来说：
  - 如果属性或索引器的声明可访问性为 public，访问器修饰符可以是 **private protected**、protected internal、internal、protected 或 private 中的任意一个。
  - 如果属性或索引器的声明可访问性为 protected internal，访问器修饰符可以是 **private protected**、internal、protected 或 private 中的任意一个。
  - 如果属性或索引器的声明可访问性为 internal 或 protected，访问器修饰符必须是 **private protected 或** private。
  - **如果属性或索引器的声明可访问性为 private protected，访问器修饰符必须是 private。**
  - 如果属性或索引器的声明可访问性为 private，则不得使用任何访问器修饰符。

-----

> 由于结构体不支持继承，结构体成员的声明可访问性不能为 protected、**private protected** 或 protected internal。

-----
