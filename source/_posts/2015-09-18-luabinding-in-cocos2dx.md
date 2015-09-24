---
layout: post
title: 基于Cocos2d-x的lua绑定机制
category: Cocos2d-x
tag: Cocos2d-x
---

## tolua++

#### tolua如何工作

[tolua++](https://github.com/LuaDist/toluapp)是[tolua](http://webserver2.tecgraf.puc-rio.br/~celes/tolua/)的扩展版本，一款能够极大地简化C/C++代码与Lua代码集成的工具。tolua基于一份简单干净的头文件，能够自动生成从Lua代码中访问C/C++特性的绑定代码，通过使用Lua API和标记方法工具，tolua来映射管理C/C++中的常量，外部变量，函数，类和方法来给Lua使用。

使用tolua通常要先建立pkg文件，里面包含对应C/C++头文件中能够被tolua++解析的声明，如常量、变量、函数以及类等。tolua++会根据pkg文件，来生成用于C/C++源文件与Lua交互的c/cpp文件，从而实现Lua中访问指定的C/C++代码。其中，无论是C函数还是C++对象的方法，都一律导出为静态函数。

#### 如何使用tolua

生成的tolua++可执行文件，支持一些可选选项，使用./tolua++ -h可进行查看。

 + 使用 __-o__ 来指定生成的c/cpp文件

	./tolua++ -o myfile.c myfile.pkg

 + 使用 __-L__ 来在优先通过都dofile()来运行一份指定的Lua文件

 	./tolua++ -o myfile.c myfile.pkg -L rule.lua

 + 使用 __-n__ 来指定导出到lua的package

 	./tolua++ -n pkgname -o myfile.c myfile.pkg

 绑定生成代码必须被编译并链接到应用程序，才能被Lua代码访问。每一个被解析的package文件代表要导出到Lua的包。默认情况，包名称就是输入文件的名称。用户可以指定一个不同的包名称。在tolua中，需要明确地初始化解析出的对应的包给Lua。在C/C++代码中我们应该调用初始化函数 __int tolua_pkgname_open(void)__ 来进行初始化。其中 _pkgname_ 代表被绑定的包的名称。

 更加详细的说明请参考[这里](http://webserver2.tecgraf.puc-rio.br/~celes/tolua/tolua-3.2.html)。

## cocos2d-x的lua绑定机制

#### 版本2.x实现

在本节主要基于quick-cocos2d-x的2.x版本来进行讲解分析。在quick根目录的lib下，Lua绑定相关的代码放置于luabinding目录下，其中build.sh用来生成绑定文件。

	#!/bin/bash
	DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
	TOLUA="$QUICK_COCOS2DX_ROOT/bin/mac/tolua++"

	cd "$DIR/"
	${TOLUA} -L "$DIR/basic.lua" -o "$QUICK_COCOS2DX_ROOT/lib/cocos2d-x/scripting/lua/cocos2dx_support/LuaCocos2d.cpp" Cocos2d.tolua

根据脚本文件内容我们不难发现，其实就是执行了quick根目录bin/mac下的tolua++可执行文件，根据Cocos2d.tolua来解析生成LuaCocosd.cpp文件用于Lua代码中访问cocos2d-x的原生接口。其中还指定优先装载了basic.lua这个Lua文件，由于basic.lua文件篇幅较长，这里先不列出。其中basic.lua中重写实现了 __post_output_hook(packag)__ 函数，该函数会在所有所有输出流写完之后执行。basic.lua通过这个函数主要进行了一些替换操作，以保证可以按照cocos2d-x自己的特殊需要修改输出的绑定生成cpp文件。

举个例子，cocos2d-x会在CCLuaValue.h中声明诸如LUA_STRING、LUA_FUNCTION和LUA_TABLE的类型，它们实际上就是int整型。为了防止tolua++将LUA_FUNCTION等解析成自定义类型，利用basic.lua中的replace函数将一些出现LUA_FUNCTION的地方都替换成了空的字符串。

正常如果我们想添加自己的c++代码并绑定给Lua使用，需要在quick-cocos2d-x中建立自己的tolua文件，该文件等同于之前的pkg文件，同样用于声明相关会被解析给Lua使用的常量、函数和类等。并建立一份build.sh脚本文件，内容如下：

	#!/usr/bin/env bash
	DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
	cd "$DIR"
	OUTPUT_DIR="$DIR"
	QUICK_COCOS2DX_ROOT="$DIR/../../.."
	MAKE_LUABINDING="$QUICK_COCOS2DX_ROOT"/bin/compile_luabinding.sh
	$MAKE_LUABINDING -d "$OUTPUT_DIR" test.tolua

其中test.tolua是我们建立的pkg文件，里面声明了一个用于解析的C++类，其中类中有两个成员函数分别如下声明：

	string gzipDecompressStringLua(const char* data, int len);
	LUA_STRING gzipCompressStringLua(const char* data, int len);

第一个gzipDecompressStringLua函数返回一个std::string给Lua使用，而第二个gzipCompressStringLua函数则返回LUA_STRING（实际上是int）给Lua层。执行刚刚新建立的build.sh，会自动生成一个cpp和h文件，里面就是根据tolua声明文件而对指定cpp文件解析生成的中间代码。首先会对Lua栈中的相关函数变量以及参数进行正确性地检查，没有问题后会根据tolua中的tolua_toXX系列函数生成调用所需的类的对象以及相关参数，最后调用c++对应的函数并获得返回值，通过tolua_pushXX系列函数将值压入Lua栈中。具体代码如下：

	static int tolua_jj_luabinding_JJZipUtil_gzipDecompressStringLua00(lua_State* tolua_S)
    {
    #ifndef TOLUA_RELEASE
        tolua_Error tolua_err;
        if (
            !tolua_isusertype(tolua_S,1,"JJZipUtil",0,&tolua_err) ||
            !tolua_isstring(tolua_S,2,0,&tolua_err) ||
            !tolua_isnumber(tolua_S,3,0,&tolua_err) ||
            !tolua_isnoobj(tolua_S,4,&tolua_err)
        )
        goto tolua_lerror;
        else
    #endif
        {
            JJZipUtil* self = (JJZipUtil*)  tolua_tousertype(tolua_S,1,0);
            const char* data = ((const char*)  tolua_tostring(tolua_S,2,0));
            int len = ((int)  tolua_tonumber(tolua_S,3,0));
    #ifndef TOLUA_RELEASE
        if (!self) tolua_error(tolua_S,"invalid 'self' in function 'gzipDecompressStringLua'", NULL);
    #endif
        {
            string tolua_ret = (string)  self->gzipDecompressStringLua(data,len);
            tolua_pushcppstring(tolua_S,(const char*)tolua_ret);
        }
        }
        return 1;
    #ifndef TOLUA_RELEASE
        tolua_lerror:
        tolua_error(tolua_S,"#ferror in function 'gzipDecompressStringLua'.",&tolua_err);
        return 0;
    #endif
    }

而第二个返回LUA_STRING的函数，生成代码类似，只不过调用c++函数并获得返回值处的代码变成了没有处理返回值以及压入Lua栈的操作，需要我们在具体函数实现中调用相应的api来将指定的值压入栈中，并可以指定压入数据的长度，这样可以避免一些数据截断的情况的发生。对应生成代码如下：

	self->gzipCompressStringLua(data,len);

回顾刚刚的build.sh，不难发现，实际上就是去调用quick-cocos2d-x的根目录下bin/compile_luabinding.sh脚本文件，该脚本又调用了bin/lib/compile_luabinding.php脚本文件，其中该目录下还有其他的php脚本文件，如compile_luabinding_functions.php文件中针对LUA_STRING等类型进行了特殊处理，与之前的cocos2d-x中原生处理LUA_FUNCTION的思路一致。

#### 版本3.x实现

在cocos2d-x 3.x版本后，针对Lua的绑定做了一些变动，开发者们不需要再针对每一个要解析绑定的cpp文件来编写pkg(tolua)文件了。在cocos2d-x/tools/tolua目录下，我们发现有许多ini为后缀的文件。根据这些ini配置文件，执行同一目录下的genbindings.py就会生成绑定代码cpp/hpp文件于cocos2d-x/cocos/scripting/lua-bindings/auto目录下。具体如何添加自定义的C++类绑定给Lua使用，请参考官方帮助文档[如何使用 bindings-generator 自动生成 lua绑定](http://www.cocos.com/doc/article/index?type=wiki&url=/doc/cocos-docs-master/manual/framework/native/wiki/how-to-use-bindings-generator/zh.md)。

关于genbindings.py脚本文件主要做了以下几件事情：

 + 检查ndk以及llvm的支持
 + 整理并将一些配置的环境变量值写入userconf.ini中
 + 遍历cmd_args列表(其中key值就是待解析ini文件的名称)，并执行cocos2d-x/tools/bindings-generator/generator.py

因此，具体的解析操作是在cocos2d-x/tools/bindings-generator/generator.py来实现的。bindings-generator下的genertor.py不单单用来处理Lua的绑定操作，也包括js的。根据分析源码，我们发现其支持集中传入参数：

 + __-s__ 设置要转换的特定部分
 + __-t__ 指定目标虚拟机，会从TARGET.yaml中搜索
 + __-o__ 指定生成C++代码的输出目录
 + __-n__ 指定输出文件的名称，默认为.ini文件中的prefix选项

在cocos2d-x/tools/bindings-generator/targets/lua目录下，放有conversions.yaml文件以及一个templates目录，根据conversions.yaml配置与templates下的模板文件，通过clang来进行语法和词法分析来生成绑定代码。   

## 相关参考

 + [YAML简介](http://www.ibm.com/developerworks/cn/xml/x-cn-yamlintro/)
 + [使用Python和Cheetah构建和扩充模板](https://www.ibm.com/developerworks/cn/opensource/os-pythcheetah/)
 + [Parsing C++ in Python with Clang](http://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)
 + [Implementing a code generator with libclang](http://szelei.me/code-generator/)
