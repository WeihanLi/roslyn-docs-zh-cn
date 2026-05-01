Co-located 核心类型
=====================

## 客户场景与背景

ECMA 335 规范指出，某些类型（例如 `System.Object` 或 `System.Int16`）可以在元数据中以整数值而非类型引用令牌的形式被引用（内置类型）。此外，规范还指出，这些类型与一小组通过令牌引用的类型之间允许进行特定转换——我们将这些类型称为"特殊类型"。最后，还有一些类型具有特殊的运行时处理方式，但与其他内置类型的引用方式不同。

场景

1. 编译器需要在给定内置类型引用的情况下找到该类型的定义。由于引用并非类型令牌，编译器需要知道包含该定义的程序集。

2. 编译器遇到同一内置类型的两个定义。

3. 编译器被要求对两个内置类型引用进行类型检查，其中一个来自使用不兼容核心类型定义编译的库。

## 设计

编译器可以假设恰好存在一个代表"核心程序集"的程序集。该核心程序集要么被引用，要么正在被编译。核心程序集不得引用任何其他程序集。

核心程序集包含平台所支持的内置类型和特殊类型子集的类型定义。

对于名称与内置类型相同但位于核心程序集之外的类型的引用，通过类型令牌进行。

仅引用内置类型的程序集 `A` 可以（根据 ECMA 规范）省略对核心库的引用。当编译器在编译第二个程序集 `B` 时进一步引用 `A` 的情况下，`A` 中对内置类型的引用由编译器根据用于编译 `B` 的核心库来解析。

平台实现可以选择定义一个仅包含内置类型和特殊类型子集的核心程序集。在此场景下，只要不直接（通过用户代码）或间接（通过编译器特性）引用该子集之外的内置类型或特殊类型，编译就能成功。这意味着，编译器虽然可以安全地假设所有内置类型和特殊类型都位于同一程序集中，但它们不一定都存在。

如果平台的后续版本引入了更多内置类型或特殊类型，必须将其添加到核心程序集中。

相应地，此类平台的运行时只需支持相应核心程序集中的子集。

co-located 类型的完整列表如下：

- `System.Object`
- `System.Enum`
- `System.MulticastDelegate`
- `System.Delegate`
- `System.ValueType`
- `System.Void`
- `System.Boolean`
- `System.Char`
- `System.SByte`
- `System.Byte`
- `System.Int16`
- `System.UInt16`
- `System.Int32`
- `System.UInt32`
- `System.Int64`
- `System.UInt64`
- `System.Decimal`
- `System.Single`
- `System.Double`
- `System.String`
- `System.IntPtr`
- `System.UIntPtr`
- `System.Array`
- `System.DateTime`
- `System.Collections.IEnumerable`
- `System.Collections.Generic.IEnumerable<>`
- `System.Collections.Generic.IList<>`
- `System.Collections.Generic.ICollection<>`
- `System.Collections.Generic.IEnumerator<>`
- `System.Collections.IEnumerator`
- `System.Nullable<>`
- `System.Runtime.CompilerServices.IsVolatile`
- `System.IDisposable`
- `System.TypedReference`
- `System.IAsyncResult`
- `System.AsyncCallback`
- `System.Collections.Generic.IReadOnlyList<>`
- `System.Collections.Generic.IReadOnlyCollection<>`

我们继续假设：在自定义特性中对 `System.Type` 的引用以枚举值方式进行，而在其他地方则作为常规引用处理。注意这与 ECMA 规范存在偏差，规范中包含了 `System.Type`。`System.Type` 和其他反射类型有意不放在核心库中。由于 `System.Type` 可以作为内置类型被引用，编译器可以假设在编译上下文中最多存在该类型的一个定义，并将存在多个定义的情况视为错误。只要没有引用，就不应接受任何定义。

依赖列表之外 API 的语言特性，不应假设此类 API 定义的具体程序集位置。编译器应使用启发式方法，或要求明确指定此类 API 的位置。
