---
layout: post
title: 在Python中使用Clang来解析C++【翻译】
category: 编程相关
tag: python
---

   [原文地址](http://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)

Clang的API发展十分迅速，其中也包括libclang和Python绑定。因此，本次推送中的示例可能不再起作用。如果想要那些基于最新的Clang的工作示例，请检查我在Github上的[llvm-clang-samples repository仓库](https://github.com/eliben/llvm-clang-samples)。

对于那些需要在Python中去解析和分析C代码的人，在发现[pycparser](https://github.com/eliben/pycparser)后一定会很兴奋。然而，如果变成是去解析C++，pycparser并不是好的解决办法。当我被问及关于在pycparser中增加支持C++的计划时，我的回答通常是 -- 没有这样的计划，你应该去别处找找。尤其地，是在[Clang](http://clang.llvm.org/)中。

Clang是一款针对C，C++和Object C的编译器前端。它是由Apple支持的一款具有自由协议的开源项目，并使用它们自己的工具。连同它的父项目 -- LLVM编译器后端，Clang渐渐成为一款gcc的强大替代品。在Clang（包括LLVM）身后有着一流的开发团队，并且它的代码在开源环境下也是C++代码中设计最好的之一。Clang的发展十分积极，密切关注着最新的C++标准。

于是当我被问关于解析C++的问题的时候，我的回答总是Clang。诚然，它还存在着一些小问题。人们喜欢pycparser因为它是Python实现的，而Clang的API接口是C++，并不是最极客并友好的语言，退一步来说。

<!-- more -->

### libclang

下面进入libclang。不太久之前，Clang的团队明智地意识到Clang并不仅仅可以用于编译器，也可以是分析C/C++/ObjC代码的工具。事实上，Apple的自研Xcode开发工具就是用Clang作为内置库来进行代码提示，交叉引用，等等。

能使Clang如此被利用的组件我们称之为libclang。它是C API编写的，并且Clang团队郑重声明保证其稳定性，允许使用者在抽象语法树（AST）的等级上去检查解析代码。

更多的技术上来讲，libclang是一个通过使用面向公众的API来对Clang进行包装的共享库，并只定义在一份C语言头文件中：clang/include/clang-c/Index.h 。

### Python绑定到libclang

libclang于 _clang/bindings/python_ 中附带了Python绑定，位于 _clang.cindex_ 模块中。这个模块依赖于 _ctypes_ 去装载动态的libclang库，并试图尽可能地去封装libclang作为Python API接口。

### 文档？

非常不幸，目前libclang的相关文档以及Python绑定的状态十分糟糕。官方文档会根据源代码而进行开发（自动生成Doxygen HTML）。此外，我所能找到的所有在线文档只有一份[演示](http://llvm.org/devmtg/2010-11/Gregor-libclang.pdf)和一些来自Clang开发邮件列表中过时的邮件信息。

好的一方面，即使你只浏览了一遍 __Index.h__ 头文件也能记住它试图要实现什么，它的API并不难懂（具体实现也是一样易懂，哪怕你对Clang的内部只熟悉一点）。另一个需要看的地方是 __clang/tools/c-index-test__ 工具，它是用来测试API接口并示例它们该如何使用。

对于Python绑定，也完全没有任何文档，除了它发布的源代码和一些相关示例。因此，我希望这篇文章能够起到帮助！

### 设置

设置使用Python绑定很简单：

 + 你的脚本需要能够找到 __clang.cindex__ 模块。所以要适当地复制它或建立 __PYTHONPATH__ 来指向它。

 + __clang.cindex__ 需要能够找到 __libclang.so__ 共享库。取决于你如何构建/安装Clang，你需要适当地拷贝它或是建立 __LD_LIBRARY_PATH__ 来指向它的位置。在Windows操作系统，则是 __libclang.dll__ 并且需要设置到 __PATH__ 环境变量中去。

准备就绪后，要去调用 __import clang.cindex__ 并开始使用。

### 简单示例

让我们开始一段简单的示例。下面的脚本使用 __libclang__ 的Python绑定来找到一个给定文件中的所有引用类型：

	#!/usr/bin/env python
	""" Usage: call with <filename> <typename>
	"""

	import sys
	import clang.cindex

	def find_typerefs(node, typename):
	    """ Find all references to the type named 'typename'
	    """
	    if node.kind.is_reference():
	        ref_node = clang.cindex.Cursor_ref(node)
	        if ref_node.spelling == typename:
	            print 'Found %s [line=%s, col=%s]' % (
	                typename, node.location.line, node.location.column)
	    # Recurse for children of this node
	    for c in node.get_children():
	        find_typerefs(c, typename)

	index = clang.cindex.Index.create()
	tu = index.parse(sys.argv[1])
	print 'Translation unit:', tu.spelling
	find_typerefs(tu.cursor, sys.argv[2])

假设我们在下面这段C++代码中调用它：

	class Person {
	};

	class Room {
	public:
	    void add_person(Person person)
	    {
	        // do stuff
	    }

	private:
	    Person* people_in_room;
	};

	template <class T, int N>
	class Bag<T, N> {
	};

	int main()
	{
	    Person* p = new Person();
	    Bag<Person, 42> bagofpersons;

	    return 0;
	}

执行去查找 __Persion__ 引用类型，我们会得到：

	Translation unit: simple_demo_src.cpp
	Found Person [line=7, col=21]
	Found Person [line=13, col=5]
	Found Person [line=24, col=5]
	Found Person [line=24, col=21]
	Found Person [line=25, col=9]

### 理解它是如何工作的

为了理解这个例子做了什么，我们需要明白它内部运作的3个层次：

 + 概念层次 -- 我们从要被解析的源代码获取的信息是什么，并且它是如何储存的；
 + libclang层次 -- __libclang__ 的正式C API，要比Python绑定代码更容易阅读，虽然只在代码中有一些备注；
 + Python绑定层次，这是我们直接调用的。

#### 创建索引并解析代码

我们需要在最开始时添加如下代码：

	index = clang.cindex.Index.create()
	tu = index.parse(sys.argv[1])

一个"index"代表一组编译和链接在一起的翻译单元。我们需要一些分组翻译单元的方法，如果试图理解它们的话。举例来说，我们可能想要找到一些定义在头文件中的引用类型，以及在一些其他的源文件中。 __Index.create()__ 调用C API函数 __clang_createIndex__。

接下来，我们使用 __Index's parse__ 方法来解析一个文件中单独的翻译单元。调用 __clang_parseTranslationUnit__，一个在C API中的关键函数。它的注释提到：

 >  这个程序是Clang C API中的主要入口，提供将一份源文件解析成一个翻译单元的功能，并且能查询API中其他的函数。

这是一个强大的函数 -- 它能可选择性地正常接受全套标识并传到命令行编译器。它返回一个封装了的 __CXTranslationUnit__ 对象，作为一个封装了Python绑定的翻译单元。这个翻译单元能够被查询，例如翻译单元的名字可以通过 __spelling__ 属性表示：

	print 'Translation unit:', tu.spelling

然而，它最重要的属性是 -- __cursor__。一个 _cursor_ 是libclang中的关键抽象，它表示一个被解析翻译单元的抽象语法树中的一些节点。在一个单一抽象下的程序中，cursor整合不同类型的实体，提供一组常见的操作，如获取它的位置和子cursor。 __TranslationUnit.cursor__ 返回一个翻译单元最高层级的cursor，作为索引它的抽象语法树的声明点。

#### 使用curosrs进行工作

Python绑定将 __libclang__ 中的cursor封装成 __Cursor__ 对象。它有许多属性，其中最有趣的包括：

 + __kind__ -- 一个枚举指定了这个curosr指向的抽象语法树节点的种类；
 + __spelling__ -- 节点的源代码名称
 + __location__ -- 被解析节点的源代码位置
 + __get_children__ -- 它的子节点

 __get_children__ 需要专门说明一下，因为这是一个在C和Python的API接口分歧上的特殊点。

 __libclang__ C API 基于访问者想法。根据给定的cursor走到抽象语法树，用户代码提供了一个针对 __clang_visitChildren__ 的回调函数。然后在一个给定的抽象语法树的所有子节点上来调用这个函数。

而另一方面的Python绑定，会封装在内部访问，通过 __Cursor.get_children__ 提供一个Python化的迭代API，返回一个指定cursor的子节点。它仍然可以通过Python来直接访问最原始的API接口，但是使用 __get_children__ 更加地方便。在我们的例子中，我们使用 __get_children__ 来递归访问一个节点的所有子节点：

 	for c in node.get_children():
    	find_typerefs(c, typename)

#### Python绑定的一些局限性

不幸的是，Python绑定还不够成熟并存在一些缺陷，因为它是一项正在进行中的工作。举一个例子，假设我们想要找到和记录这个文件中的所有函数调用：

	bool foo()
	{
	    return true;
	}

	void bar()
	{
	    foo();
	    for (int i = 0; i < 10; ++i)
	        foo();
	}

	int main()
	{
	    bar();
	    if (foo())
	        bar();
	}

让我们写下这段代码：

	import sys
	import clang.cindex

	def callexpr_visitor(node, parent, userdata):
	    if node.kind == clang.cindex.CursorKind.CALL_EXPR:
	        print 'Found %s [line=%s, col=%s]' % (
	                node.spelling, node.location.line, node.location.column)
	    return 2 # means continue visiting recursively

	index = clang.cindex.Index.create()
	tu = index.parse(sys.argv[1])
	clang.cindex.Cursor_visit(
	        tu.cursor,
	        clang.cindex.Cursor_visit_callback(callexpr_visitor),
	        None)

这次直接使用 __libclang__ 的访问API，结果如下：

	Found None [line=8, col=5]
	Found None [line=10, col=9]
	Found None [line=15, col=5]
	Found None [line=16, col=9]
	Found None [line=17, col=9]

既然被记录的位置是正确的，那为什么节点的名字会是 __Node__ 呢？仔细研究过libclang的代码后发现，我们不应该来打印 _spelling_ ，而应该是 _display_ 。在C API中它意味着 __clang_getCursorDisplayName__ 而不是 __clang_getCursorSpelling__ 。但是，Python绑定并没有暴露 __clang_getCursorDisplayName__ 接口！

然而，我们不应该就此放弃。Python绑定的相关源代码十分地直截了当，并且很容易通过 __ctypes__ 来在C API中开放额外的函数。只要在 __bindings/python/clang/cindex.py__ 中添加如下几行：

	Cursor_displayname = lib.clang_getCursorDisplayName
	Cursor_displayname.argtypes = [Cursor]
	Cursor_displayname.restype = _CXString
	Cursor_displayname.errcheck = _CXString.from_result

现在我们可以使用 __Cursor_displayname__，在脚本中通过 __clang.cindex.Cursor_displayname(node)__ 来代替 __node.spelling__ 。这样可以得到我们想要的输出结果：

	Found foo [line=8, col=5]
	Found foo [line=10, col=9]
	Found bar [line=15, col=5]
	Found foo [line=16, col=9]
	Found bar [line=17, col=9]

 _更新提示（06.07.2011）_：来自这篇文章的灵感，我向Clang项目中提交了关于开放 __Cursor_displayname__ 接口的代码，也修复了一些在Python绑定中的其他问题。它已经在Clang的核心开发版本134460中被提交，并且现在应该在主干trunk上可用。

#### libclang的一些局限性

综上所述，Python绑定的一些局限性相对容易客服。自从 __libclang__ 提供了一个简单易行的C API，这仅仅就是适当使用 __ctypes__ 结构来开放暴露附加函数的问题而已。对于任何有些Python经验的人来说，这都不是一个大问题。

然而一些 __libclang__ 本身的局限性，例如，假设我们想要找到一段代码中的所有返回语句，结果发现通过当前 __libclang__ 的API是办不到的。大致浏览 __Index.h__ 就会发现其原因。

 __enum CXCursorKind__ 枚举了我们在 __libclang__ 中可能遇到的cursor节点的种类。下面是这部分相关的代码：

	/* Statements */
	CXCursor_FirstStmt                     = 200,
	/**
	 * \brief A statement whose specific kind is not exposed via this
	 * interface.
	 *
	 * Unexposed statements have the same operations as any other kind of
	 * statement; one can extract their location information, spelling,
	 * children, etc. However, the specific kind of the statement is not
	 * reported.
	 */
	CXCursor_UnexposedStmt                 = 200,

	/** \brief A labelled statement in a function.
	 *
	 * This cursor kind is used to describe the "start_over:" label statement in
	 * the following example:
	 *
	 * \code
	 *   start_over:
	 *     ++counter;
	 * \endcode
	 *
	 */
	CXCursor_LabelStmt                     = 201,

	CXCursor_LastStmt                      = CXCursor_LabelStmt,

忽略用于正确性测试的占位符 __CXCursor_FirstStmt__ 和 __CXCursor_LastStmt__ ，这唯一需要注意的是lable语句。所有其他的声明语句将会被 __CXCursor_UnexposedStmt__ 所替代。

去理解这个缺陷的原理，有建设性的思考是 __libclang__ 的主要目标。目前，API主要用于集成开发环境当中，我们想要知道一切类型和引用符号，但不用明确关心我们看到的声明或是语句的种类。

值得庆幸的是，从Clang的邮件开发列表讨论组中可以收集到这种局限性并非蓄意而为之。在 __libclang__ 中根据需要添加的东西，显然没有一个需要 __libclang__ 来辨别声明种类的不同，因此没有人增加这个特性。如果它对某人来说足够重要，他可以随时在邮件列表中提出一个修补补丁。特别是，这个具体的缺陷（缺少声明种类）是很容易被解决的。看看在 __libclang/CXCursor.cpp__ 中的 __cxcursor::MakeCXCursor__ ，很明显找到这些"种类"是如何生成的：

	CXCursor cxcursor::MakeCXCursor(Stmt *S, Decl *Parent,
                                CXTranslationUnit TU) {
  		assert(S && TU && "Invalid arguments!");
  		CXCursorKind K = CXCursor_NotImplemented;

  		switch (S->getStmtClass()) {
  		case Stmt::NoStmtClass:
    		break;

  		case Stmt::NullStmtClass:
  		case Stmt::CompoundStmtClass:
  		case Stmt::CaseStmtClass:

  		... // many other statement classes

  		case Stmt::MaterializeTemporaryExprClass:
    		K = CXCursor_UnexposedStmt;
    		break;

  		case Stmt::LabelStmtClass:
    		K = CXCursor_LabelStmt;
   	 		break;

  		case Stmt::PredefinedExprClass:

  		.. //  many other statement classes

  		case Stmt::AsTypeExprClass:
    		K = CXCursor_UnexposedExpr;
    		break;

  		.. // more code
  	}

这是在 __Stmt.getStmtClass()__ 中的一个简单而巨大的switch语句，并且仅对 __Stmt::LabelStmtClass__ 有一种非 __CXCursor_UnexposedStmt__ 类型。所以建议添加的"类型"不要是特别重要的：

 1. 向 __CXCursorKind__ 添加另一个枚举值，在 __CXCursor_FirstStmt__ 和 __CXCurosr_LastStmt__ 之间；
 2. 在 __cxcursor::MakeCXCursor__ 中的switch语句中添加另一份条件case以识别适合的类和返回值种类；
 3. 在Python绑定中暴露相关枚举值。

### 结论

希望这篇文章介绍 __libclang__ 的Python绑定的文章能够起到帮助。尽管这些组件缺乏外部文档，但它们被很好的编写和注释，并且源代码足够的浅显易懂。

记住这些积极发展、并包装在十分强大的C/C++/ObjC的解析引擎里的API，显得十分重要。仅代表个人观点，Clang是当下最新的C++解析库中最好的选择，没有之一。

在 __libclang__ 本身以及Python绑定中存在着一些美中不足的小限制。这些都是最近相对除了Clang之外， __libclang__ 的附带结果，毕竟它还是一个非常年轻的项目。

幸运的是，希望我这篇文章能够告诉你们这些限制并非极度难以解决的。仅需要少量Python和C的专业知识，就能够扩展Python绑定，并且一点对Clang的理解也可以增强奠定 __libclang__ 本身。另外， __libclang__ 仍然在积极地发展当中。我十分确信这些API将会随着时间的推移而持续改进，并且限制也会越来越少。

