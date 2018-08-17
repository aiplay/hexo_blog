---
layout: post
title: 使用PhysX构建简单的物理系统
date: 2018-08-15
category: 游戏开发
tag: physics
---

### PhysX物理引擎

PhysX是一套由AGEIA公司开发的物理运算引擎(后被NVIDIA收购)，简而言之，就是令虚拟世界中的物体运动符合真实世界的物理定律，以使游戏更加富有真实感。PhysX可以由CPU计算，但其程序本身在设计上还可以调用独立的浮点处理器（例如GPU和PPU）来计算，也正因为如此，它可以轻松完成像流体力学模拟那样的大计算量的物理模拟计算。

目前最火热的两款商业游戏引擎Unreal和Unity都采用PhysX作为它们的物理引擎来模拟物理计算，并且PhysX已经于2014年宣布开源，现在在Github上就能免费下载全部源码。不过需要注册NVIDIA的开发者账号，并注册加入其在Github上的组织才能下载源码。

具体方法请参考[这里](https://developer.nvidia.com/physx-source-github)。

<!-- more -->

### 物理系统

所谓物理系统，就是针对物理引擎做出的进一步封装，方便开发者添加和使用需要的物理特性。如果你的游戏引擎是基于组件式的架构的话，那么一个简单的物理系统可能需要以下两个基本的物理组件：__Rigidbody__、__Collider__。并且需要一个PhysicsManager来管理物理引擎的初始化、物理模拟和相关资源的释放等。其中每个Collider组件都可以添加物理材质，它仅仅是用来设定物理碰撞时的动静态摩擦力以及反弹系数等数值。

其实物理系统还可以拆分诸多个子系统，比如__布料系统__、__布娃娃系统__、__物理粒子系统__和__Vehicle车辆系统__等。但这里不会做过多的详细介绍，接下来仅会对Rigidbody和Collider的实现原理进行分析说明。

具体的物理系统架构：
![01](http://7xlmp2.com1.z0.glb.clouddn.com/2018-08-15%E7%89%A9%E7%90%86%E7%B3%BB%E7%BB%9F.png)

### Rigidbody组件

#### 概要

在游戏场景中，可能会有许许多多的游戏元素，但哪些需要去表现一些物理特性（也就是被物理引擎去模拟计算）呢？这就需要用到Rigidbody组件，例如在Unity中，如果一个GameObject包含Rigidbody组件，那么就可以认为它会受到物理世界的模拟影响，比如自由落体运动。

本质上就是在Rigidbody中维护着一个PxRigidActor类型的指针，它会被添加进PhysX的物理场景中，由物理引擎去模拟计算每一个固定时长帧下的刚体状态变化。在渲染的时候，我们再根据物理引擎计算得出的Rigidbody位置、旋转信息，附加设置到对应GameObject的Transform组件上即可。

下面是PxRigidActor类的继承关系图：
![02](http://7xlmp2.com1.z0.glb.clouddn.com/2018-08-15-physx_actor_tree.png)

由上图可以发现在PxRigidActor下面，还会继续派生出PxRigidDynamic和PxRigidStatic来。实际上在Rigidbody组件中，会更多地用到PxRigidDynamic的特性，比如质量、阻力、速度等等。而PxRigidStatic更多的是为那些只有Collider组件而没有Rigidbody组件的静态物体所使用，这类静态物体不会受任何力的影响，但会参与到碰撞当中去，并能够给其他有Rigidbody组件的物体以影响。

#### 质量属性

一个dynamic actor所需的质量属性包括：质量，转动惯量和质心。在PhysX中，计算质量属性的方式是调用__PxRigidBodyExt::updateMassAndInertia()__函数，也可以通过__PxRigidBodyExt::setMassAndUpdateInertia()__来改变刚体的质量分布。

每个PxRigidActor可以动态地维护1到n个__PxShape__，每当shape的增加或减少，我们都应去更新刚体的质量分布以获得更加逼真的物理表现。而每个PxShape的指针，我们会放到Collider组件中去维护，在之后会做详细地介绍。

#### 运动学刚体

还是拿Unity举例子，当一个Rigidbody组件被设置为IsKinematic时，表示该刚体不再受任何力的作用，但是可以对其他动态刚体施加作用力。刚体实现设置为Kinematic很简单，只需要调用:

    PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true);

将刚体设置为Kinematic通常是为了希望可以人为地去改变物体的Transform而不是通过物理引擎的模拟计算结果。但人为修改Transform时，应注意区分__PxRigidDynamic::setKinematicTarget()__和__PxRigidActor::setGlobalPose()__。当使用setGlobalPose()时，仅仅会将actor移动至合适的位置，而不和其他物体发生交互。特别注意的是，Kinematic的刚体使用setGlobalPose()并不会推开经过路径上的其他动态刚体。

如果两个刚体都是Kinematic的，则不会产生碰撞效果，但是可以请求获得企图发生碰撞的两个物体的相关碰撞信息， __PxSceneFlag::eENABLE_KINEMATIC_PAIRS__ 或 __PxSceneFlag::eENABLE_KINEMATIC_STATIC_PAIRS__ 被设置上即可。

### Collider组件

#### 概要

Collider组件主要是为了封装PxShape，之前也有说过，每一个物理世界中的物体(PxRigidActor)可能会对应多个PxShape，但是Collider组件只会维护一个PxShape，表示这个碰撞体的指定形状。由于组件化的灵活机制，可能某一个子GameObject只有Collider组件而没有Rigidbody组件，因此要向上查找其Parent GameObject是否持有Rigidbody组件，如果有的话则将该Collider维护着的PxShape指针attach到找到的Rigidbody组件持有的PxRigidActor上（记得需要更新质量分布属性）。

具体架构如下图：
![03](http://7xlmp2.com1.z0.glb.clouddn.com/2018-08-15-relation_colliders_pxshape.png)

#### Shapes

在PhysX中，使用PxShape来描述刚体的空间范围和碰撞属性，当我们创建PxShape的时候，需要先构建PxGeometry和PxMaterial，构建PxGeometry的派生类对象，常见的包括：__Box__、__Sphere__、__Capsule__、__Convex Meshes__ 和 __Triangle Meshes__ 等。其中TriangleMesh类型的几何体不支持Simulation Shape附加在动态刚体上，除非该刚体被配置为Kinematic。根据不同的Geometry类型，我们可以封装成不同的组件并继承自Collider，一一对应，具体包括：__BoxCollider__、__SphereCollider__、__CapsuleCollider__ 和 __MeshCollider__ (统一管理Convex和Triangle)。

每个子Collider实现中，要去管理对应Geometry所需的特征数据，如SphereCollider得去维护Sphere Geometry的半径等，剩下的具体计算只需要交给PhysX。无论PxShape对应什么形状，其具体Transform都是位于PxRigidActor坐标系下的，因此我们只能修改PxShape的相对位置。世界坐标系下的Transform信息，应交给PxRigidActor来关心。动态刚体的运动轨迹最好由PhysX计算完成，而静态刚体则可以通过调用 __setGlobalPose()__ 函数来进行设置。

> __特别注意__ :
> _1. 在构建或更新Geometry的时候，需要考虑Collider组件对应的GameObject缩放系数_
> _2. 在PhysX中，Capsule是沿X轴方向水平拉长的，如果想做成类似Unity那种沿Y轴方向的效果，需要手动旋转90度_

#### 物理材质

在现实世界中，不同的物体发生碰撞总会产生不同的运动效果。比如汽车在冰面上行驶和在水泥地上行驶的摩擦阻力肯定是不同的。在PhysX中封装了PxMaterial来描述不同物体在物理世界中的个体差异。在构建PxShape时，除了PxGeometry外，PxMaterial也是必须的。

创建一个PxMaterial很简单，只要调用静态函数__PxMaterial::createMaterial()__并传入__静态摩擦力系数__、__动态摩擦力系数__ 和 __反弹系数__ 就可以了。同时PxMaterial还支持设置摩擦力结合模式和反弹结合模式，可根据发生碰撞的两个物体的物理材质系数，取平均值、最大值和最小值等。

### 碰撞检测

一般的碰撞检测算法，都要分2个阶段：__Broad Phase__ 和 __Narrow Phase__ 。其中Broad Phase主要用于构建场景中碰撞盒的BVH(Bounding Volume Hirerarchy)。通过构建BVH，来配对(潜在)碰撞盒。而在Narrow Phase中，会对Broad Phase已确定的碰撞盒进行二次检测，确定最终的碰撞结果。在PhysX中，Broad Phase主要的算法包括 __SAP(Sweep And Prune)__ 和 __MBP(Multi Box Pruning)__。具体关于SAP的介绍，可以参考[这里](https://en.wikipedia.org/wiki/Sweep_and_prune)。

使用MBP算法，同时会引入Regions的概念。Regions就是世界坐标系下的AABB包围盒体积空间，该空间外的所有物体不会进行碰撞检测，理想情况下应该覆盖到整个游戏空间，并且最大数量不要超过256。出于对性能的考虑，尽量不要让不同的Regions相互重叠，两个Regions的AABB仅仅触碰并不算重叠。

创建物理场景时，可以指定一个 __PxSimulationFilterCallback__ 的回调函数指针。该回调将会在场景中所有的shape pair的包围盒第一次相交时执行，并根据回调的返回值来确定接下来的行为。每一个PxShape对象中都持有一个 __PxFilterData__ 类型的成员变量，用128bit的数据来指定跟Collision Filter有关的信息，这些信息都会在PxSimulationFilterCallback回调中以PxFilterData类型的参数传递进来，同时回调参数还包括一块指定大小的内存，可以用来传递更多的数据信息，需要在构建PxSceneDesc时指定 __filterShaderData__ 和 __filterShaderDataSize__ 。

利用这一灵活的特性，我们可以方便地实现__动态__设置刚体是否支持 __CCD(Continuous Collision Detection)__ 和__按层过滤碰撞机制__。这两个功能可以有效地提高物理模拟的性能，减少一些不必要的计算消耗。每一个PxFilterData只包含4个int类型的成员(word0~word3)，我们可以先用word3来表示发生碰撞的对应shape是否支持CCD，具体的设置需要放到Collider组件中，查找其对应依附的Rigidbody组件并获取其CCD支持状态。具体代码如下:

    PxU32 filterFlags0 = (filterData0.word3 & 0xFFFFFF);
    PxU32 filterFlags1 = (filterData1.word3 & 0xFFFFFF);
    bool ccdCondition0 = (filterFlags0 & CCD_MODE::DYNAMIC) && !(filterFlags1 & CCD_MODE::OFF);
    bool ccdCondition1 = (filterFlags0 & CCD_MODE::NORMAL) && (attributes1 & PxFilterObjectType::eRIGID_STATIC);
    if (!(k0 && k1) && (ccdCondition0 || ccdCondition1))
    {
        pairFlags |= PxPairFlag::eSOLVE_CONTACT;
        pairFlags |= PxPairFlag::eDETECT_CCD_CONTACT;
    }
    pairFlags |= PxPairFlag::eDETECT_DISCRETE_CONTACT;

其中pairFlags是回调的引用参数，用于输出碰撞对的状态。

至于按层过滤碰撞机制，实现方式类似，不过要用到filterShaderData。在filterShaderData中存储着一个长度为32的uint32_t数组，里面每一个int的每一个bit用来表示不同层之间的碰撞关系(0表示不支持碰撞，1表示支持碰撞)。因此这就限制了层的最大上限只能是32，实际上Unity的实现也是如此。与此同时，我们可以使用PxFilterData.word0来标识对应Collider的隶属层的索引，通过位运算来计算出是否支持产生碰撞，不支持则直接函数返回 _PxFilterFlag::eSUPPERESS_ 。代码片段如下：

    PxU32 shapeGroup0 = (filterData0.word0 & 0xFFFFFFFF);
    PxU32 shapeGroup1 = (filterData1.word0 & 0xFFFFFFFF);
    uint32_t* groupCollisionFlags = (uint32_t*)constantBlock;
    if ((groupCollisionFlags[shapeGroup0] & (1 << shapeGroup1)) == 0)
    {
        return PxFilterFlag::eSUPPRESS;
    }

其中constantBlock就是指在构建物理场景时传入的层碰撞状态数组。

### 固定步长刷新

渲染，表现的只是游戏时间中的一瞬间，通常不需要关心距离上次渲染过去了多少时间，因此会放到变时步长刷新中。但是物理模拟运算不同，它需要一个合理的固定步长刷新机制，来使得物理表现更加逼真、稳定。也就是说，我们以固定时间步长来更新模拟物理计算，但渲染的时间点却是随机的，让我们看看时间线：

![04](http://7xlmp2.com1.z0.glb.clouddn.com/2018-08-15-timeline-with-update.png)

如上图所见，时间线上端的“更新”表示一次物理模拟运算，它们之间的间隔是相同而紧凑的，但是渲染发生的位置是不固定的并且通常频率会低于更新。因此，我们并不能保证总在更新的时间点进行渲染，观察下第三次和第四次渲染，它们就是发生在两次更新之间。试想一颗子弹横穿屏幕，首次更新它在屏幕的左侧，而第二次更新它将移动到屏幕的右侧，如果渲染在两次更新之间进行，并且只根据前一次更新结果的话，子弹会被渲染到屏幕的左侧，而我们更希望子弹出现在屏幕的中间某一位置。

为了更逼真的物理运动表现，或减少一些不必要的抖动问题，就要求我们去做一些额外的差值运算。在Unity的刚体组件中，提供了设置Interpolate来选择差值方式，其中Interpolate表示根据前一固定帧来进行差值计算，Extrapolate则表示预测下一固定帧的位置来进行差值计算。

所需的差值计算方式可以交给Rigidbody组件来记录，除了Interpolate和Extrapolate之外，当然也可以什么都不做，即None。假设用于渲染的变时步长回调为update()，用于物理模拟运算的定时步长回调为fixedUpdate()。在每次fixedUpdate回调中执行物理场景的simulate()函数之前，遍历物理场景下的所有动态刚体，如果刚体为Interpolate模式，要把对应刚体的位置和旋转记录下来。等执行update()回调的时候，Interpolate模式下，会根据之前记录的位置、旋转信息与当前刚体的信息进行差值计算。而Extrapolate模式下，在update()回调中会根据动态刚体的velocity和angularVelocity来预测计算出当前渲染时间点的刚体位置和旋转数据来。

### 参考

[PhysX SDK](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/Index.html)
[PhysX API](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/apireference/files/index.html)
[Unity Manual 3D Physics Reference](https://docs.unity3d.com/Manual/Physics3DReference.html)
