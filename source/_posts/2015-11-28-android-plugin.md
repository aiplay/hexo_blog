---
layout: post
title: Android插件机制研究分析
date: 2015-11-28
category: Android
tag: Android
---

### 初识插件化

早在Android 2.3的年代，依旧存在着著名的 __65535__ 问题，即应用项目内的最大方法数不能超过65535个，否则无法编译通过。随着应用业务逻辑的不断发展，无论如何优化代码，也很快会遇到这个瓶颈。多数的解决方案，是通过拆分并装载多个dex字节码文件来规避这个问题。但对于结构复杂、业务逻辑繁重的应用来说，仅仅拆分dex文件是远远不够的，因此，插件化的这个概念也随之应运而生。

所谓 __插件化__ ，我们可以理解为将一个单独的apk文件，通过某种机制，使其可以在未安装的状态下直接运行。这样一来，我们便可以将主应用的多个业务模块拆分出来，单独编译成不同的apk文件，不仅可以减轻主应用的体积，还能使业务逻辑模块化，降低模块间的耦合。

简要概括下来，插件化主要包括以下三个优点：

 + 动态更新
 	当某个业务模块需要改进更新的时候，不用再升级安装庞大的宿主应用，仅仅自动下载对应的轻量插件便可以，并且无需用户去安装。这对于产品的迭代优化，紧急Bug的修复都有着很大的帮助。

 + 可定制化
 	插件化在降低模块耦合的同时，也为应用的定制给予了极大的便利。针对不同的发布渠道或是运营需要，宿主应用能够在不修改代码的情况下从容地进行定制，大大降低了发布成本。

 + 并行协作开发
 	对于大型应用，独立的插件可以在不同的团队之间并行进行开发，缩短项目进程，提高开发效率。

<!-- more -->

### 插件机制的具体实现

目前在Android平台实现插件机制的方法主要包括两种： _宿主注册代理组件，反射绑定维护插件对应组件_ 和 _利用动态代理技术hook系统以管理运行插件_ 。目前使用这两种技术并在github上拥有2k+ star的开源项目有 [dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk) 和 [DroidPlugin](https://github.com/Qihoo360/DroidPlugin)，下面来分别介绍一下。

#### [dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)

这个项目是由百度资深Android开发工程师[任玉刚](http://my.csdn.net/singwhatiwanna)发起的，具体结构层次图(来自其说明文档)如下：

![dynamic-load-apk结构层次图](http://7xlmp2.com1.z0.glb.clouddn.com/dl-frame.png)

其实现原理就是先在宿主程序中注册好用于代理的Activity和Service组件，并在其onCreate生命周期函数中对需启动的插件利用反射进行相关绑定以及初始化的操作。启动插件的Activity或是Service组件，实际上就是启动宿主本地的代理组件，并在执行代理组件生命周期函数的同时，绑定执行对应插件组件的声明周期函数。

具体相关源码的分析可以参考这篇[文章](http://a.codekk.com/detail/Android/FFish/DynamicLoadApk%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)。

在使用dynamic-load-apk框架开发插件时，存在着一些限制，如插件内的Activity或Service需继承框架库封装的DLBasePluginActivity或DLBasePluginService（插件项目要外部引入dl-lib.jar），而不能直接继承Activity和Service，因为需要插件库去帮你管理插件组件的声明周期。并且，插件apk也不能独立运行与Android设备上，对于开发调试与插件移植还是存在着一定的困难。但是其结构清晰简单，并且有着丰富的说明文档，足够满足一些简单的插件实现，帮助实现插件化的管理工作。

#### [DroidPlugin](https://github.com/Qihoo360/DroidPlugin)

这是奇虎360开源的一款插件机制框架，并成功运用于360手机助手软件上帮助其实现一些插件化的功能。

DroidPlugin实现原理不同于之前的dynamic-load-apk，它利用Java的动态代理技术将Android Framework层hook住并替换成自己实现的各种Manager，这样就模拟出来了一个伪Android系统，可以达到欺上瞒下的功效------让插件apk文件认为自己被安装过了，让系统认为插件apk确实被安装过了。

下面是DroidPlugin的实现原理图：

![DroidPlugin原理图](http://7xlmp2.com1.z0.glb.clouddn.com/DroidPlugin-frame.png)

虽然DroidPlugin框架hook住了整个framework并使其插件可以“浑水摸鱼”来运行于系统之上，却仍然存在着一些限制缺陷。

 + 无法在插件中发送具有自定义资源的Notification；

 + 无法在插件中注册一些具有特殊Intent Filter的Service、Activity、BroadcastReceiver、ContentProvider等组件以供Android系统、已经安装的其他APP调用；

 + 缺乏对Native层的Hook，使其诸如基于cocos2d-x或是u3d开发手机游戏不能很好地作为插件运行。

但是其优点也显而易见，插件apk不依赖任何lib库，并能够直接安装运行与Android系统之上，几乎做到了让apk文件在不安装的情况运行起来。
