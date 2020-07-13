---
title: 在 Ubuntu 上安装 Enlightenment Desktop
date: 2016-08-23 06:46:36
categories:
- ['应用实践','笔记']
tags:
- 'Linux'
- 'Desktop Environments'
---
最近知道有个桌面环境叫 Enlightenment Desktop ，看上去很酷炫，打算安装试用一下。

官方网站是 [enlightenment.org](www.enlightenment.org) ，上面有详细的介绍以及安装方式，这里只介绍如何在 Ubuntu 16.04 上安装。

首先添加其官方软件源，据官方说是因为 Ubuntu 上的源的 Enlightenment 已经很旧了，所以推荐使用新版。

	sudo add-apt-repository ppa:niko2040/e19

然后更新并安装桌面的本体
	
	sudo apt-get update
	sudo apt-get install enlightenment terminology connman

其中 enlightenment 是本体； terminology 是其终端模拟器，很酷炫的说； connman 是网络管理器，建议安装。

然后登出即可切换至 Enlightenment 19 了。