**本文档列出了 Roslyn 3.0（Visual Studio 2019）相对于 Roslyn 2.*（Visual Studio 2017）的已知重大更改。**

1. 以前，引用程序集在发出时会包含嵌入式资源。在 Visual Studio 2019 中，嵌入式资源不再发出到引用程序集中。
  参见 https://github.com/dotnet/roslyn/issues/31197
