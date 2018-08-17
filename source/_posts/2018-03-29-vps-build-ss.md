---
layout: post
title: 购买vps搭建ss服务教程
date: 2018-03-29
category: 网络相关
tag: Network
---

### 购买vps

伴随着十*大的胜利召开，很多ss服务提供商都相继中招倒戈了，其中也包括我用了两年多的【一支红杏】。最后痛定思痛，决定还是自己折腾一下vps靠谱些，虽然价格上会稍微贵一点，但肯定会更稳定，而且用来搭建新的博客系统也是一个不错的选择。

在[VPS中文网](https://www.vpszh.com/)上发现墙裂推荐了几款vps服务商，最后选择了  __搬瓦工__ ，虽然性价比可能不是最高的，但是貌似针对中国线路做了很多优化，这样在速度上可能会更有优势吧。至于到底该选择哪家vps服务商，还是仁者见仁智者见智，主要还是得根据自己的切实需求来选择。

<!-- more -->

进入[__搬瓦工官网__](https://bwh1.net/)，可能大多数人会和我一样点击页面右上方的【Register】按钮来注册，注意的是里面的注册信息中，国家和邮箱一定一定不要乱填，其他的倒还都好。但是当你辛辛苦苦填完注册信息，点击【Submit】按钮的时候，却会提示你验证码输入失败，因为此时的你可能根本看不到验证码的存在，主要是因为搬瓦工用的是Google的验证码服务，所以你懂的。这就回归到了是先有鸡还是先有蛋的纠结上了，其实大可以直接选择你想要的vps主机配置，加到购物车然后进行__checkout__，进入结算页的时候，会再次弹出注册信息的填写，而且不再有验证码的限制了，这样就可以成功注册并购买vps服务。

话说回来，现在很多vps服务商都支持支付宝付款，个人觉得这点还是十分方便的，不用再去折腾master信用卡之类的支付方式。

#### 配置vps

等注册购买成功后，点击右上方的【Client Area】，并选择【My Service】，会进入到如下界面：

![01](http://7xlmp2.com1.z0.glb.clouddn.com/bwh_my_service.png)

点击右侧的【KiwiVM Control Panel】按钮，会进入到你的vps管理界面：

![02](http://7xlmp2.com1.z0.glb.clouddn.com/bwh_kiwivm_panel.png)

下面有显示你的ip地址和端口号，如果需要重装系统的话，记得要先【stop】服务器，重装系统成功后，再重新【start】。我装的是Centos 6 x86_64 bbr，它会自动帮你开启bbr加速，就不用折腾其他的加速器了。

#### 搭建ss服务

以前在KiwiVM的管理面板左侧，就有一键安装ss服务的选项，但是现在没有了，需要自己来搭建。在网上搜了很多资料，其实很好搭建，就用pip来安装ss就可以了，省时省力不容易出错。

安装成功后在/etc目录新建shadowsocks.json文件，内容如下：

    {
    "server":"你的ip",
    "port_password":{
    "端口1":"密码1",
    "端口2":"密码2",
    "端口3":"密码3"
    },
    "local_address":"127.0.0.1",
    "local_port":1080,
    "timeout":300,
    "method":"rc4-md5"
    }

编写保存好该json文件后，在命令行运行如下命令：

    ssserver -c /etc/shadowsocks.json -d start

如果要关闭服务的话，则运行：

    ssserver -c /etc/shadowsocks.json -d stop

服务启动成功后，只要在你的客户端连接vps的ip，并输入指定的端口和密码，以及加密方式就大功告成。

最后晒一张YouTube上一个1080的高清视频截图：

![03](http://7xlmp2.com1.z0.glb.clouddn.com/vps_youtube_1080.png)
