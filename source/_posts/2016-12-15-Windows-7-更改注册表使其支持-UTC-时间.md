---
title: Windows 7 更改注册表使其支持 UTC 时间
date: 2016-12-15 06:27:50
categories:
- '应用实践'
- '笔记'
tags:
- 'Windows'
---
默认情况下 Windows 7 是不支持硬件 UTC 时间的，所以导致从 Linux 上切换过来之后时间就不对了。

解决方法是，通过修改注册表使其开启 UTC 时间支持。

打开 regedit.exe (打开开始菜单在搜索框里输入这个名字就会出现，或者到 C:\Windows 下面找），找到 HKEY\_LOCAL\_MACHINE -> SYSTEM -> CurrentControlSet -> Control -> TimeZoneInformation ，新建一个 RealTimeIsUniversal 的键， 类型为 DOWRD64 ，重启即可。也可以直接复制下面的文本保存到文本文件，更改后缀名为 .reg 后双击。

	Windows Registry Editor Version 5.00
	
	[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation] “RealTimeIsUniversal”=dword:00000001
	
