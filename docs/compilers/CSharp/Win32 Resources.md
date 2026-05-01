# Win32 资源

编译器接受以下本机资源类型：
 - 通过 /win32manifest 提供的文件中的任意数据（假定为文本）。这些数据作为资源添加到输出二进制文件中，DLL 类型为 #24，编号为 #2；EXE 类型为 #24，编号为 #1。
 - 通过 /win32res 提供的格式定义在[此处](http://msdn.microsoft.com/en-us/library/ms648007(VS.85).aspx)的资源文件（.RES）。
 - 通过 /win32icon 提供的格式描述在[此处](http://msdn.microsoft.com/en-us/library/ms997538.aspx)的图标。
	
同时指定 /win32icon 或 /win32manifest 以及 /win32res 是一个错误。用户应该要么提供所有资源（通过 /win32res），要么提供部分资源（使用其他开关），但编译器不会将用户提供的 .RES 文件的内容与单独资源合并。

除非指定了 /win32res，否则版本资源将添加到输出中。在缺少影响版本号的属性的情况下，编译器会构建默认版本号。
