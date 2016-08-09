---
layout: post
title: Sol2使用教程
category: Lua
tag: lua binding
---

### Sol2简介

Sol是一个用于C++绑定Lua脚本的库，仅由头文件组成，方便集成，并提供了大量易用的API接口，可以便利地将Lua脚本与C++代码绑定起来，而不必去关心如何使用那些晦涩的Lua C API。正如其作者所言，Sol的目的就是提供极其简洁的API，并能高效到与C语言媲美，极大地来方便人们使用。

Sol支持Lua的绝大多数版本，包括 5.1、5.2、5.3和LuaJit等，但由于代码中用到了许多C++11/14特性，因此编译时需要编译器支持C++14标准。

具体使用以及介绍详情，请前往[Github仓库](https://github.com/ThePhD/sol2)和[Sol2主页](http://sol2.readthedocs.io/en/latest/index.html)。

### 基础使用

从Sol的[Github仓库](https://github.com/ThePhD/sol2)clone下代码后，我们发现其目录下很多test开头的cpp/hpp文件，这些文件里面有着大量的Sol的使用示例以及各种特性的展示，而在example目录下的cpp文件都仅仅是一些最基础的使用示例。为了方便测试和体验Sol，你也可以自己建立一些自己的test.cpp文件，首先你要在源文件中include引用sol.hpp头文件，这样才能使用Sol提供的接口。而在使用gcc编译的使用，需要指定关联头文件的路径，可以使用如下命令：

	g++ test.cpp -Isolpath/single/sol -llua -std=c++14

其中solpath是你Sol2的具体路径，在Sol2的项目目录下，有一个single/sol/sol.hpp头文件，这个头文件集成了所有的相关代码到一起，所以编译时 -I 后仅指定这一个路径就可以了，同时要保证你的gcc编译器支持C++14标准。

#### 绑定第一个类

	#include "stdio.h"
	#include "sol.hpp"

	class A
	{
	public:
		A () = default;
		A (int a) { mA = a; };
		virtual ~A() = default;
		void foo()
		{
			printf("foo excute!\n");
		}
	private:
		int mA;
	}

	int main(void)
	{
		sol::state lua;
		// 打开lua中的相关库，如table、string等
		lua.open_libraries();
		lua.new_usertype<A>
		(
			"A", // Lua中识别的类名
			sol::constructors<sol::types<int>>(),
			"foo", &A::foo // 绑定成员函数foo
		)

		// 执行一段Lua程序
		lua.script(" local a = A.new(); a:foo(); ")
	}

上述程序中，将一个简单的类A绑定到了Lua代码中，在Lua代码中可以创建一个a对象并调用其成员函数foo()。其中，A类除了默认构造函数外，还有一个带参数的构造函数，需要使用sol::constructors来封装一个，并通过sol::types<>模板来指定其参数列表。

#### 设置全局函数

我们可以将一个C++的函数设置到Lua中作为全局函数调用，代码如下:

	sol::state lua;
	lua.set_function("func", func); // func变量为C++函数地址

#### 重载函数绑定

如果一个类中的某个成员函数被重载了，即函数名及返回值相同，参数列表不同。我们可以这样绑定:

	sol::state lua;
	lua.new_usertype<A>
	(
		"A", // sol会帮你构造一个默认构造函数
		"foo", sol::overload(sol::resolve<void(int)>(&A::foo),
			sol::overload(sol::resolve<void(int, int)>(&A::foo)))
	)

假设A类包含两个名字都叫做foo的成员函数，都没有返回值(void)，一个是需要一个int作为参数，另一个是需要两个int作为参数，当Lua代码中调用foo函数的时候，Sol会根据你传递的参数来判断调用对应哪个绑定的C++函数。

#### 将枚举映射到Lua的全局table中

有时候在C++中会定义许多枚举，为了方便Lua代码中可以方便地使用对应的枚举类型，我们可以将其绑定到一个Lua全局table中。

	enum Color
	{
		RED,
		GREEN,
		BLUE
	};
	sol::state lua;
	lua["Color"] = lua.create_table_with("RED", RED, "GREEN", GREEN, "BLUE", BLUE);
	lua.script("Color.RED")

上段代码中，我们实际上构建了一个全局table，名字叫做Color，并将table的key对应的value设置成枚举Color中的对应项，这样就实现了在Lua使用某个枚举里的内容。

### 进阶使用

#### 将Lua的table传到C++中

很多时候，我们需要将一个table传到C++层来使用，Sol为我们封装了sol::table类型来表示Lua中的table，假设我们定义了一个C++函数，其参数就是sol::table。

	void foo(sol::table t)
	{
		// t 表示一个从Lua层传来的table
	}

	sol::state lua;
	lua.set_function("foo", foo);

	lua.script("local t = { 1, 2, 3, 4 }; foo(t); ")

通过上边Lua代码的调用，就一个内容是“1, 2, 3, 4”的table传给了C++层。

#### 遍历Lua传来的table

由于Lua中只有table这一种数据结构，它既可以表示数组也可以表示HashMap的结构，因此遍历获取sol::table中的内容显得更加复杂。Sol重载了[]让我们可以很方便地去获取table中的值，可以传递索引或key值。

	void foo(sol::table t)
	{
		// lua table { 1, 2, 3, 4 }
		sol::object obj = t[1];
		int first = obj.as<int>();  // first is 1

		// lua table { a = "a", b = 2 }
		sol::object a = t["a"];
		sol::object b = t["b"];
		sol::type atype = a.get_type(); // atype 等于 sol::type::string
		sol::type btype = b.get_type(); // btype 等于 sol::type::number
		std::string astr = a.as<std::string>(); // a
		int bnum = b.as<int>();  // 2
	}

通过[]得到的table中的值并不能直接当成number或是string来使用，它实际上是sol::proxy代理类型，我们可以将其隐式转换成sol::object类型，并通过 __as<>()__ 来获取其在Lua中代表的值对应的C++类型。

如果我们并不清楚这个Lua层传入的table是怎样的结构，可以采取遍历并判断对应key、value的类型，从而得到我们实际想要的值，代码如下：

	void foo(sol::table t)
	{
	    for (auto& kv : t)
	    {
	        sol::object key = kv.first;
	        sol::object val = kv.second;
	        printf("key type : %d, val type : %d\n", key.get_type(), val.get_type());
	        switch (key.get_type())
	        {
	            case sol::type::number:
	                // 这是一个数组
	                break;
	            case sol::type::string:
	                // 键值对table
	                break;
	        }
	        switch (val.get_type())
	        {
	            case sol::type::number:
	                break;
	            case sol::type::string:
	                break;
	            case sol::type::table:
	                break;
	        }
	    }
	}

#### 让代码更加健壮

#### 隐式传递Lua状态机