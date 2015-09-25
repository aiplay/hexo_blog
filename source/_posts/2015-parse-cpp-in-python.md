---
layout: post
title: 在Python中使用Clang来解析C++【翻译】
category: 编程相关
tag: python
---

   [原文地址](http://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)

Clang的APIs发展十分迅速，其中也包括libclang和Python绑定。因此，本次推送中的示例可能不在起作用。想要那些基于最新的Clang的工作示例，请检查我在Github上的[llvm-clang-samples repository仓库](https://github.com/eliben/llvm-clang-samples)。

那些需要在Python中去解析和分析C代码的人，在发现[pycparser](https://github.com/eliben/pycparser)后一定会很兴奋。然而，当任务变成是去解析C++，pycparser并不是好的解决办法。当我被问及关于在pycparser中支持C++的计划时，我的回答通常是 -- 没有这样的计划，你应该去别处找找。尤其地，是在[Clang](http://clang.llvm.org/)中。

Clang是一款针对C，C++和Object C的编译器前端。它是由Apple支持的一款具有自由协议的开源项目，并使用它们自己的工具。连同它的父项目 -- LLVM编译器后端，Clang渐渐成为一款gcc的强大替代品。在Clang（包括LLVM）身后有着一流的开发团队，并且它的代码在开源环境下也是C++代码中设计最好的之一。Clang的发展十分积极，密切关注着最新的C++标准。

于是当我被问关于解析C++的问题的时候，我的回答总是Clang。诚然，它还存在着一些小问题。人们喜欢pycparser因为它是Python实现的，而Clang的API接口是C++，并不是最极客并友好的语言，退一步来说。

### libclang

下面进入libclang。不太久之前，Clang的团队明智地意识到Clang并不仅仅可以用于编译器，也可以是分析C/C++/ObjC代码的工具。事实上，Apple的自研Xcode开发工具就是用Clang作为内置库来进行代码提示，交叉引用，等等。

能使Clang如此被利用的组件我们称之为libclang。它是C API编写的，并且Clang团队郑重声明保证其稳定性，允许使用者在抽象语法树（AST）的等级上去检查解析代码。

更多的技术上来讲，libclang是一个通过使用面向公众的API来对Clang进行包装的共享库，并只定义在一份C语言头文件中：clang/include/clang-c/Index.h 。

### Python绑定到libclang

libclang于 _clang/bindings/python_ 中附带了Python绑定，位于 _clang.cindex_ 模块中。这个模块依赖于 _ctypes_ 去装载动态的libclang库，并试图尽可能地去封装libclang作为Python API接口。
