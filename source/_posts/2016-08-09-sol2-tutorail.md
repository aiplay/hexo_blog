---
layout: post
title: Sol2使用教程
date: 2016-08-09
category: Lua
tag: lua binding
---

### Sol2简介

Sol是一个用于C++绑定Lua脚本的库，仅由头文件组成，方便集成，并提供了大量易用的API接口，可以便利地将Lua脚本与C++代码绑定起来，而不必去关心如何使用那些晦涩的Lua C API。正如其作者所言，Sol的目的就是提供极其简洁的API，并能高效到与C语言媲美，极大地来方便人们使用。

Sol支持Lua的绝大多数版本，包括 5.1、5.2、5.3和LuaJit等，但由于代码中用到了许多C++11/14特性，因此编译时需要编译器支持C++14标准。

具体使用以及介绍详情，请前往[Github仓库](https://github.com/ThePhD/sol2)和[Sol2主页](http://sol2.readthedocs.io/en/latest/index.html)。

### 基础使用

从Sol的[Github仓库](https://github.com/ThePhD/sol2)clone下代码后，我们发现其目录下很多test开头的cpp/hpp文件，这些文件里面有着大量的Sol的使用示例以及各种特性的展示，而在example目录下的cpp文件都仅仅是一些最基础的使用示例。为了方便测试和体验Sol，你也可以自己建立一些自己的test.cpp文件，首先你要在源文件中include引用sol.hpp头文件，这样才能使用Sol提供的接口。而在使用gcc编译的时候，需要指定关联头文件的路径，可以使用如下命令：

	g++ test.cpp -Isolpath/single/sol -llua -std=c++14

其中solpath是你Sol2的具体路径，在Sol2的项目目录下，有一个single/sol/sol.hpp头文件，这个头文件集成了所有的相关代码到一起，所以编译时 -I 后仅指定这一个路径就可以了，同时要保证你的gcc编译器支持C++14标准。

<!-- more -->

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

上述程序中，将一个简单的类A绑定到了Lua代码中，在Lua代码中可以创建一个a对象并调用其成员函数foo()。其中，A类除了默认构造函数外，还有一个带参数的构造函数，需要使用sol::constructors进行封装，并通过sol::types<>模板来指定其参数列表。

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

通过上边Lua代码的调用，使得一个内容是“1, 2, 3, 4”的table传给了C++层。

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

由于在Lua中类型的概念很弱，函数定义的参数并不会指定具体类型，这导致当我们想调用某个C++绑定过来的接口传递table的时候，很有可能有意或无意传递了一个nil过去。如果该函数在C++中指定的类型是sol::table，而在Lua传递一个nil值会发生很多不可控的错误，因此需要我们针对这种情况特殊处理以保证代码更加健壮。

无论是sol::object还是sol::table、sol::function等类型，都继承于sol::reference类。因此，可以通过定义在sol::reference中的 __valid()__ 函数即可判断当前对象是否可用。sol::table类型也可以通过 __empty()__ 来判断当前table是否包含内容。

	void foo(sol::table t)
	{
		if (! t.valid() || t.empty())
			return;
		// 执行到这表明t并不为nil或是空表，可以继续代码逻辑
	}

#### 隐式传递Lua状态机

虽然Sol帮我们封装了很多简单易用的接口，但实际开发过程中难免还是需要自己去和Lua状态机打交道。在之前代码经常出现的sol::state中，我们可以通过 __lua_state()__ 接口来获取lua_State指针。

同时，Sol封装了一个sol::this_state结构体，可以方便地将其隐式地转成lua_State指针。在绑定到Lua中的C++函数里，可以在参数列表的任意位置添加sol::this_state声明，而在Lua代码中调用该函数时则可以忽略该参数，Sol会自动帮你将Lua代码执行时的状态机作为sol::this_state来进行传递。

	void foo(sol::table t, sol::this_state l)
	{
		// 向Lua栈中push一个字符串
		lua_pushstring(l, "hello");
		// 构建一个sol::object对象
		sol::object obj = sol::make_object(l, t);
	}

	int main(void)
	{
		sol::state lua;
		lua.set_function("foo", foo);
		// 执行Lua代码时，调用foo函数仅传递了一个table作为参数
		lua.script("local t = { a = 1, b = 2 }; foo(t);");
	}

#### C++与Lua的返回值

##### 接收来自Lua的多个返回值

	int main () {
        sol::state lua;

        lua.script("function f (a, b, c) return a, b, c end");

        std::tuple<int, int, int> result;
        result = lua["f"](1, 2, 3);
        // result == { 1, 2, 3 }
        int a, int b;
        std::string c;
        sol::tie( a, b, c ) = lua["f"](1, 2, "bark");
        // a == 1
        // b == 2
        // c == "bark"
	}

##### 返回多种类型给Lua

在C++代码中，无法做到像Lua那样可以灵活地返回各种类型的值，但是我们可以利用sol::object来封装成不同的“Lua类型”返回给Lua层。

	sol::object fancy_func (sol::object a, sol::object b, sol::this_state s) {
        sol::state_view lua = s;
        if (a.is<int>() && b.is<int>()) {
                return sol::make_object(lua, a.as<int>() + b.as<int>());
        }
        else if (a.is<bool>()) {
                bool do_triple = a.as<bool>();
                return sol::make_object(lua, b.as<double>() * ( do_triple ? 3 : 1 ) );
        }
        return sol::make_object(lua, sol::nil);
	}

	sol::state lua;

	lua["f"] = fancy_func;

	int result = lua["f"](1, 2);
	// result == 3
	double result2 = lua["f"](false, 2.5);
	// result2 == 2.5

	// call in Lua, get result
	lua.script("result3 = f(true, 5.5)");
	double result3 = lua["result3"];
	// result3 == 16.5
