本地函数
===============

此功能支持在块作用域内定义函数。

TODO: _编写规范_


语法文法
==============

此文法以与当前规范文法的差异形式表示。

```diff
 declaration_statement
    : local_variable_declaration ';'
    | local_constant_declaration ';'
+   | local_function_declaration
    ;

+local_function_declaration
+   : local_function_header local_function_body
+   ;

+local_function_header
+   : local_function_modifier* return_type identifier type_parameter_list?
+        '(' formal_parameter_list? ')' type_parameter_constraints_clauses
+   ;

+local_function_modifier
+   : 'async'
+   | 'unsafe'
+   ;

+local_function_body
+   : block
+   | arrow_expression_body
+   ;
```

本地函数可以使用封闭作用域中定义的变量。当前实现要求在本地函数内读取的每个变量在其定义点处都已明确赋值，就好像在本地函数的定义点处执行该本地函数一样。此外，本地函数定义必须在任何使用点之前已"执行"过。

在对此进行了一些实验之后（例如，无法定义两个相互递归的本地函数），我们已修订了对明确赋值工作方式的期望。修订后（尚未实现）的规则是：在本地函数中读取的所有局部变量必须在本地函数每次调用时都已明确赋值。这实际上比听起来更微妙，还有大量工作需要完成才能实现。一旦完成，你就可以将本地函数移到其封闭块的末尾。

新的明确赋值规则与推断本地函数的返回类型不兼容，因此我们可能会移除对推断返回类型的支持。

除非将本地函数转换为委托，否则捕获是通过值类型帧完成的。这意味着使用带捕获的本地函数不会产生任何 GC 压力。
