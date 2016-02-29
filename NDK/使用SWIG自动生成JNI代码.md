# 使用SWIG自动生成JNI代码

## 什么是SWIG


*SWIG:*简化的包装器和接口生成器，Simplified Wrapper And Interface Generator

SWIG不是Android或者Java的专用工具，它是一个可以生成许多其他编程语言代码的、广泛使用的工具。

SWIG是一个编译时软件开发工具，能生成将用C/C++编写的原生模块与包括Java在内的其他编程语言进行联接的必要代码。它不仅仅是一个代码生成器，还是一个接口编译器。他不定义新的协议，也不是一个组件框架或者一个特定的运行时库。SWIG把接口文件看做输入，并生成必要的代码在Java中展示接口，从而让Java能够理解原生代码中的接口定义。SWIG不是一个存根生成器；它产生将要被编译和运行的代码。

## 在MAC上安装SWIG

SWIG 网站没有提供MAC OS X平台的安装包，需要用Homebrew包管理器下载并安装SWIG(需要安装Homebrew)。

然后执行命令:`brew install swig`,然后执行`swig -version`验证安装