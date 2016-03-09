---
layout: post
title: 在Mac系统上搭建OpenGL开发环境
category: openGL
tag: openGL
---

### 什么是OpenGL

( _以下来自[维基百科](https://zh.wikipedia.org/wiki/OpenGL)_ )

OpenGL英文全称 _Open Graphics Library_ ，是一个开放图形库，定义了一个跨编程语言、跨平台的API规范，它用于生成二维、三维图像。这个接口由近三百五十个不同的函数调用组成，用来从简单的图形比特绘制复杂的三维景象。而另一种程序界面系统是仅用于Microsoft Windows上的Direct3D。OpenGL常用于CAD、虚拟实境、科学可视化程序和电子游戏开发。

OpenGL的高效实现（利用了图形加速硬件）存在于Windows，很多UNIX平台和Mac OS。这些实现一般由显示设备厂商提供，而且非常依赖于该厂商提供的硬件。开放源代码库Mesa是一个纯基于软件的图形API，它的代码兼容于OpenGL。但是，由于许可证的原因，它只声称是一个“非常相似”的API。

OpenGL规范由1992年成立的OpenGL架构评审委员会（ARB）维护。ARB由一些对创建一个统一的、普遍可用的API特别感兴趣的公司组成。根据OpenGL官方网站，2002年6月的ARB投票成员包括3Dlabs、Apple Computer、ATI Technologies、Dell Computer、Evans & Sutherland、Hewlett-Packard、IBM、Intel、Matrox、NVIDIA、SGI和Sun Microsystems，Microsoft曾是创立成员之一，但已于2003年3月退出。

### 一些常见的库

#### GLEW

[GLEW](https://github.com/nigels-com/glew)是一个OpenGL扩展库，在现代OpenGL中，API函数是在运行时确定的，而不是编译期。GLEW可以帮助我们在运行时加载OpenGL API。

它的编译十分简单，通过Github下载源代码后，在终端进入glew目录，并在终端下依次执行如下命令即可。

	make extensions
	sudo make install
这样在我们的系统目录/usr/local/include和/usr/local/lib下会生成GLEW的相关头文件与静态库文件，之后创建项目时我们会用到。

#### GLFW

[GLFW](https://github.com/glfw/glfw)是一款开源、跨平台的图形窗口管理库，不仅可以用来管理窗口，还支持读取输入，处理事件等。

在Mac系统上我们需要使用[CMake](https://cmake.org/)来编译GLFW。首先在Github上clone下GLFW的源代码，然后在glfw目录下新建立一个build-glfw目录(其实这个目录建立在什么地方都可以)，然后打开CMake图形软件的界面入下图

![01](http://7xlmp2.com1.z0.glb.clouddn.com/2016-01-08-mac-config-opengl-01.png)

点击 _Configure_ 按钮，选择 __unix makefile__ 选项，如果列表中有红色提示的话，再点击一下 _Configure_ 按钮，接着点击 _Generate_ 按钮。没有出错的话，在终端中cd进入build-glfw目录，执行 __sudo make install__ 。同样地，会在系统目录/usr/local/include和/usr/local/lib下会生成GLFW的相关头文件与静态库文件。

#### GLM

[GLM](https://github.com/g-truc/glm)是一款仿照GLSL语言开发的C++图形软件数学库，它只包含C++的hpp头文件。与GLFW一样，它也可以通过Cmake来在Mac系统中编译，操作步骤类似，唯一不同的是，由于GLM仅仅包含头文件，所以编译不会生成静态库文件，只会生成/usr/local/include/glm目录。

#### SOIL

[SOIL](http://www.lonesock.net/soil.html)是一个跨平台的图片加载库，它支持加载多种图片格式，生成OpenGL的纹理。下载好该库的代码后，需要自己再编译一下。进入projects/makefile目录，新建立 _obj_ 目录，并修改makefile文件，在CXXFLAGS增加"-m64"选项，即 __CXXFLAGS = -O2 -s -Wall -m64__ 。然后在makefile目录执行 _make_ 和 _sudo make install_ ，将lib目录新生成的libSOIL.a库文件增加到xcode的链接库中。

### 使用Xcode建立项目

打开Xcode，建立一个 __Command Line Tool__ 项目。在Build Settings中找到Search Paths选项，在Header Search Paths中加入 __/usr/local/include/GLFW__ 、 __/usr/local/include/glm__ 、 __/usr/local/include__ 和 __/usr/local/include/GL__ ，在Library Search Paths中添加 __/usr/local/lib__ ，记住最右面要选择成recursive的，具体效果如下图显示。

![02](http://7xlmp2.com1.z0.glb.clouddn.com/2016-01-08-mac-config-opengl-02.png)

接着在Build Phases中的Link Binary With Libraries选项中添加我们要链接的库，如下图所示。

![03](http://7xlmp2.com1.z0.glb.clouddn.com/2016-01-08-mac-config-opengl-03.png)

具体包括的系统链接有

	CoreFoundation.framework
	Carbon.framework
	GLUT.framework
	OpenGL.framework
	Cocoa.framework
	IOKit.framework
	CoreVideo.framework

而libGLEW.a和libglfw3.a是我们之前编译好并安装到/usr/local/lib下的静态库。由于系统权限原因，我们可以进入终端输入命令 __open /usr/local/lib__ 来通过finder打开该目录，并将libGLEW.a和libglfw3.a拖拽到Xcode的Link Binary With Libraries中即可。

接下来，你便可以在main.cpp中include相应的头文件了，晒一段简单的创建窗口的代码。

	#include <glew.h>
	#include <glfw3.h>

	int main(int argc, const char * argv[]) {
    	GLFWwindow* window;
    	if (!glfwInit())
        	return -1;
    	window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    	if (!window)
    	{
        	glfwTerminate();
        	return -1;
    	}
    	glfwMakeContextCurrent(window);
    	while (!glfwWindowShouldClose(window))
    	{
        	glfwSwapBuffers(window);
        	glfwPollEvents();
    	}
    	glfwTerminate();
    	return 0;
	}