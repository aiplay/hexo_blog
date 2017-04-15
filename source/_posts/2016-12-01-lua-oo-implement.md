---
layout: post
title: Lua中实现面向对象
date: 2016-12-01
category: Lua
tag: lua
---

### 元表与元方法

在Lua中并没有真正意义上的面向对象的概念，但通过其强大的数据结构table类型，我们可以模拟出面向对象的情境来。在这之前，需要先介绍一下Lua中的元表与元方法。

元表其实本质上仍是一个table，它为用户提供了自定义行为的途径。在元表中很很多以两个下划线为前缀的键名，我们称为“事件名”，而这个键对应的函数则叫做“元方法”。在元表的键中，除了用于运算操作符、连接操作符、取长操作符和比较操作符之外，还有三种比较常见的键:

    __index : 取下标操作，用于访问表中的域
    __newindex : 赋值操作，增改表中的域
    __call : 函数调用

其中\_\_index在面向对象的实现中十分重要，下面举个例子说明一下\_\_index是如何工作的。当访问一个table中的某个属性时，Lua首先会查找该table中是否包含传入的键，如果找了对应的键，则将对应的值返回。但是如果没有找到键呢，那么Lua会接着去查找该表的元表（当然如果这个表没有元表的话，直接返回nil)，如果元表中包含\_\_index键，则会执行其对应的元方法，若\_\_index对应的是table，则会__递归性__地对这个table的元表中的\_\_index进行查找，代码如下:

``` lua
local t = {}
t.a -- is nil
local tt = {}
setmetatable(tt, {
    __index = { a = 3 }
})
tt.a -- is 3
```

上述代码中，tt表的定义并没有a属性，但是我们为它设置了一个元表，在元表中\_\_index对应的table中定义了a的值等于3，因此当我们访问tt.a的时候就会取到3了。这个特性，也可以应用到面向对象的情境上。

<!-- more -->

### 创建一个类

当创建一个类时，通常需要包含一个构造函数，并且能够通过new来创建一个该类的实例。实际上每次创建的类的实例对象，本质上仍然是一个table，但通过setmetatable来将类的table设置进去，这样实例就能访问到类的方法与属性了。具体代码如下:

``` lua
-- 封装一个创建类的函数
function class()
    local cls = {}
    cls.__index = cls

    -- 提供new接口来创建实例
    function cls.new(...)
        local instance = setmetatable({}, cls)
        -- 执行构造函数
        instance:ctor(...)
        return instance
    end

    return cls
end

-- 使用实例
local A = class()

function A:ctor(name)
    self.name_ = name
end

function A:printName()
    print(self.name_)
end

-- 创建类A的实例化对象a，并将传入的"a"通过ctor构造函数设置给self.name_
local a = A.new("a")
-- 执行打印a的name_属性
a:printName()

```

### 更复杂的类，继承

继承是面向对象编程中一个很重要的特性，而所谓的继承，就是子类能够对父类进行扩展，支持在子类中调用父类的方法。倘若子类没有某个属性或函数，可以直接使用父类的。在Lua中要实现继承的特性，本质上就是要去递归查找元表，不断地向上检索，直到找到对应的属性和方法，下面我们需要完善一下刚刚的class函数，令父类可以作为参数传递进去。

``` lua
function class(super)
    local cls = {}
    if super == nil then
        cls = { ctor = function() end }
    else
        cls = setmetatable({}, super)
        cls.super = super
    end

    cls.__index = cls

    function cls.new(...)
        local instance = setmetatable({}, cls)
        instance:ctor(...)
        return instance
    end

    return cls
end
```

下面让我们看看该如何在Lua中使用继承机制:

``` lua
Base = class()

function Base:ctor(name)
    print("Base ctor")
    self.name_ = name
end

function Base:printName()
    print("Base printName ", self.name_)
end

Sub = class(Base)

function Sub:ctor(name)
    self.super.ctor(self, name)
    print("Sub ctor")
    self.name_ = name
end

function Sub:printName()
    self.super.printName(self)
    print("Sub printName ", self.name_)
end

function Sub:more()
    print("Sub more")
end

local sub = Sub.new("sub")
sub:printName()
sub:more()

-- 打印结果
Base ctor
Sub ctor
Base printName  sub
Sub printName  sub
Sub more
```

在上述代码中，子类Sub实现的函数中可以通过self.super来访问父类对应的函数或属性。假设如果子类Sub没有实现printName函数，在执行sub:printName()的时候则会去调用其父类Base的printName函数。

### 继承来自C++绑定的userdata

在quick-cocos2d-x中，更加巧妙地封装了class函数，使得可以在Lua中去继承C++绑定的诸如CCNode等类的userdata。

使用class函数传入的super参数，如果类型是function或super.\_\_ctype为1，表示该类根父类为C++绑定的userdata。superType是“table”，表示上级父类仍是一个Lua构造的类。首先会遍历super表，依次将对应键值赋给cls表。superType是“function”，表示直接继承着就是C++类，此时传入的参数super必须为function类型，该函数的返回值就是对应的userdata，同时会将super(此时是一个function)赋值给cls.\_\_create，方便之后获取userdata。

在设置好\_\_cname类名和\_\_ctype标识后，我们再来看看new函数。利用cls.\_\_create(...)来得到父类的userdata，遍历cls并将内容深度拷贝给instance，执行构造函数后并返回instance。以下是具体实现的代码:

``` lua
    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
            cls.ctor = function() end
        end

        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    ...
```

上述代码剩余部分，是处理继承Lua构建类的流程，原理与之前的示例代码一致，不再重复贴出。
