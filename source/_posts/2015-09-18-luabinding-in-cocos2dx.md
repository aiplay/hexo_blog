---
layout: post
title: 基于Cocos2d-x的lua绑定机制
category: Cocos2d-x
tag: Cocos2d-x
---

### tolua++

在Cocos2d-x游戏引擎中，使用[tolua++](https://github.com/LuaDist/toluapp)工具来进行C/C++和Lua的绑定集成。首先建立pkg文件，里面包含对应C/C++头文件中能够被tolua++解析的声明，如常量、变量、函数以及类等。tolua++会根据pkg文件，来生成用于C/C++源文件与Lua交互的c/cpp文件，从而实现Lua中访问指定的C/C++代码。其中，无论是C函数还是C++对象的方法，都一律导出为静态函数。


