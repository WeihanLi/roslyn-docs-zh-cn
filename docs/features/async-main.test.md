主要测试领域：
* 签名的接受与拒绝
* 方法体的生成

# 签名的接受与拒绝

## 单一 Main 方法
只包含单个"main"方法的类

### 单个合法 main
* 过去：正常
* 新 7：正常
* 新 7.1：正常

### 单个 async (void) main
* 过去：错误（Async 不能作为 main）
* 新 7：错误（需要更新才能正常工作）
* 新 7.1：正常

### 单个 async (Task) main
* 过去：错误（未找到入口点），警告（签名错误，不能作为入口点）
* 新 7：错误（需要更新才能正常工作）
* 新 7.1：正常

### 单个 async (Task&lt;int&gt;) main
* 过去：错误（未找到入口点），警告（签名错误）
* 新 7：错误（需要更新才能正常工作）
* 新 7.1：正常

## 多个 Main 方法
包含多个 main 的类

### 多个合法 main
* 过去：错误
* 新 7：错误
* 新 7.1：错误

### 单个合法 main，单个 async (void) main
* 过去：错误（入口点不能标记为 async）
* 新 7：错误（此处有新错误？）
* 新 7.1：错误（此处有新错误？"void 不是可接受的 async 返回类型"）

### 单个合法 main，单个合法 Task main
* 过去：正常（警告：task main 签名错误）
* 新 7：正常（此处有新警告）
* 新 7.1：正常（此处有新警告？）

### 单个合法 main，单个合法 Task&lt;int&gt; main
* 过去：正常（警告：task main 签名错误）
* 新 7：正常（此处有新警告）
* 新 7.1：正常（此处有新警告？）

# 方法体的生成

* 检查 IL 以确认正确的代码生成。
* 确保属性正确应用于合成的 main 方法。

## Task 和 Task<T> 上的损坏方法

* 缺少 `GetAwaiter` 或 `GetResult`
* `GetAwaiter` 或 `GetResult` 缺少所需签名
* 形状正确，但缺少类型

## 缺少 Task 或 Task<T>

这将在入口点检测期间被捕获，应产生绑定错误。

## 缺少 Void 或 int

如果可以找到 Task，但找不到 void 或 int，则编译器应能优雅处理。

# Vlad 测试计划

## 编译器 API 的公共接口
  - 不适用？
 
## 一般功能
- 允许 `Task` 和 `Task<int>` 返回类型。是否要求 `async`？
- 多个有效/无效的异步候选项。
- 成功时（void 和 int）以及发生异常时的进程退出码
- `/main:<type>` 命令行开关仍可正常使用
 
## 旧版本、兼容性
- langver
- 当 async 和常规 Main 都可用时，分别使用旧/新编译器版本。
- async void Main。使用旧/新 langver。另有适用的 Main 存在时。
 
## 类型和成员
- 访问修饰符（public、protected、internal、protected internal、private），static 修饰符
- 参数修饰符（ref、out、params）
- STA 属性
- 分部方法
- 命名参数和可选参数
- `Task<dynamic>`
- 类 Task 类型
  
## 调试
- F11 正常工作
- 单步执行 Main 方法
