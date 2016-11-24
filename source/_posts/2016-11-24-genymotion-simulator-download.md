---
layout: post
title: Genymotion模拟器下载失败的解决方案
category: 网络相关
tag: Android
---

## 模拟器下载

[Genymotion](www.genymotion.com)是一款出色、高效的Android模拟器，其免费版就足以满足一名Android开发者的日常需求，使用之后令人爱不释手。然而，当下载不同版本的Android模拟器的时候，总会发生 __Connection timeout occurred__ 之类的错误，虽然可以断点续传，但还是造成了诸多不便。

解决办法就是拼接出所需下载的url地址，然后通过迅雷等下载工具进行下载，亲测速度虽然仍不是很快，但其稳定性是要好于使用浏览器下载的。在Mac操作系统中，Genymotion的模拟器文件会下载到 __~/.Genymobile/Genymotion/ova__ 目录中，里面会有你要下载的模拟器文件的全称。然后根据如下规则拼接url：

	http://dl.genymotion.com/dists/版本/ova/文件名

以下载7.0的模拟器为例，拼接后的url字就是：

	http://dl.genymotion.com/dists/7.0.0/ova/genymotion_vbox86p_7.0_160912_085006.ova

在下载完成后，将ova文件替换至 __~/.Genymobile/Genymotion/ova__ 即可。

<!-- more -->

## hosts修改

可能由于公司网络的原因，最近访问[Genymotion](www.genymotion.com)或[GitHub](www.github.com)时速度特别慢。于是google了一下，可能是CDN被墙了的缘故。浏览器打开 [ip.chinaz.com](ip.chinaz.com) ，输入访问慢的dns域名，转换生成ip地址。进入终端输入 “sudo vim /etc/hosts” 打开hosts文件，编辑加入ip地址和dns域名即可。如下：

	104.16.215.67   www.genymotion.com
	151.101.36.133  assets-cdn.github.com
	192.30.253.113  github.com

修改hosts文件后如果没有生效，建议重启浏览器或终端输入以下命令重启网络服务：

	sudo dscacheutil -flushcache
	sudo killall -HUP mDNSResponder

如果有长期翻墙访问google的需求，建议使用[老D博客](laod.cn)提供的hosts列表，会经常更新，如果访问不了再使用最新的hosts即可，网址 [https://laod.cn/hosts/2016-google-hosts.html](https://laod.cn/hosts/2016-google-hosts.html)。