---
title: Javascript 入门课程笔记
date: 2016-08-18 06:18:27
categories:
- '笔记'
- '技术'
tags:
- 'Javascript'
---
本篇笔记是学习了 [Javascript 入门课程](http://www.imooc.com/learn/36) 后的笔记

课程首先讲了Javascript的基本语法：

- 使用\<script>\</script>标签对声明Javascript代码片段
- 使用外部引入方式\<script src="filename.js">\</script>引入Javascript函数和代码片段
- //注释和/**/注释
- if和else关键字控制流程
- 使用var关键字声明变量
- 使用function关键字声明函数

在示例代码中也看见有with关键字的使用，声明在with(){}范围内的代码与哪个对象有关，使得一系列代码脚本省略了很多字。

然后讲一些互动有关的函数

 - 用document.write输出字符到此代码所在的地方
 - 用alert函数弹出警告对话框
 - 用confirm函数弹出确认对话框
 - 用prompt函数弹出输入对话框
 - 用window.open函数弹出新窗口
 - 用window.close函数关闭窗口

其中，window.open是有参数列表的:  `window.open([URL], [窗口名称], [参数字符串])`。

**URL** 代表要打开的连接，可选，如果留空则不打开任何文档。
 
**窗口名称** 表示新窗口的名称，可选。有以下注意事项：
 
 1. 该名称由字母、数字和下划线字符组成。
 2. "\_top"、"\_blank"、"\_selft"具有特殊意义的名称。
       - \_blank：在新窗口显示目标网页
       - \_self：在当前窗口显示目标网页
       - \_top：框架网页中在上部窗口中显示目标网页
 3. 相同 **name** 的窗口只能创建一个，要想创建多个窗口则 name 不能相同。
 4. **name** 不能包含空格

**参数字符串** 可选，设置窗口参数，各参数用逗号隔开。参数参照以下表格

<table>
	<tr>
		<th>参数</th>
		<th>值</th>
		<th>说明</th>
	</tr>
	<tr>
		<td>top</td>
		<td>Number</td>
		<td>窗口离屏幕顶部距离</td>
	</tr>
	<tr>
		<td>left</td>
		<td>Number</td>
		<td>窗口离屏幕左侧距离</td>
	</tr>
	<tr>
		<td>height</td>
		<td>Number</td>
		<td>窗口高度</td>
	</tr>
	<tr>
		<td>width</td>
		<td>Number</td>
		<td>窗口宽度</td>
	</tr>
	<tr>
		<td>menubar</td>
		<td>yes | no</td>
		<td>是否有菜单栏</td>
	</tr>
	<tr>
		<td>toolbar</td>
		<td>yes | no</td>
		<td>是否有工具栏</td>
	</tr>
	<tr>
		<td>scrollbar</td>
		<td>yes | no</td>
		<td>是否有滚动条</td>
	</tr>
	<tr>
		<td>status</td>
		<td>yes | no</td>
		<td>是否有状态栏</td>
	</tr>
</table>

window.close是关闭当前窗口，如果获取了某个窗口对象，则可以调用其close把它关闭。

接着讲操作节点

- 通过id获取元素：document.getElementById();
- 通过节点的innteHTML属性修改节点内容
- 通过节点的style属性修改样式：
	- style.color修改颜色
	- style.hight, style.width修改宽和高
	其它的属性同理
- 通过节点的display属性显示或者隐藏元素
	- display = "none" 隐藏元素
	- display = "block" 显示元素
- 通过节点的className属性改变节点的类名，从而应用对应类的样式