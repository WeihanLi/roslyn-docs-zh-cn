# Async Task Main
## [dotnet/csharplang 提案](https://github.com/dotnet/csharplang/blob/main/proposals/async-main.md)

## 技术细节

* 除了 `void` 和 `int` 之外，编译器还必须识别 `Task` 和 `Task<int>` 作为有效的入口点返回类型。
* 编译器必须允许在返回 `Task` 或 `Task<T>`（但不是 void）的 main 方法上放置 `async`。
* 编译器必须生成一个垫片方法 `$EntrypointMain`，该方法模仿用户定义的 main 的参数。
  * `static async Task Main(...)` -> `static void $EntrypointMain(...)`
  * `static async Task<int> Main(...)` -> `static int $EntrypointMain(...)`
  * 用户定义的 main 和生成的 main 之间的参数应完全匹配。
* 生成的 main 的主体应该是 `return Main(args...).GetAwaiter().GetResult();`
