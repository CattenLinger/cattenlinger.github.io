---
title: 慕课网《Javascript 进阶篇》编程挑战通过笔记
date: 2016-08-25 07:03:18
categories:
- '笔记'
tags:
- 'JavaScript'
---
当时我一步步把《Javascript 进阶篇》做完之后，一如既往来到编程挑战章节，当我看到要写一个模拟选项卡切换的小程序之后，我整个人都不好了。好难啊！！感觉之前的白学了！！

![](http://img.mukewang.com/53bd0e9e000163d203390200.jpg)

orz 之后我看了一下别人的代码。然后滚去看CSS去了。。。

（回来了）

好了当我重新打开这个章节的时候我还是懵逼的，于是只好继续参考别人的代码。。。然后我从CSS样式开始着手。

这个其实上面是一个无序列表元素，里面有几个列表项目包裹着三个文本标签。

	<ul>
		<li>房产</li>
		<li>家居</li>
		<li>二手房</li>
	</ul>
	
然后下面是一个类似面板的东西，参考别人代码发现其实是三个面板，只是用CSS样式隐藏掉了

	<div class="hidden">
		<ul>
			<li>内容写在这里</li>
			<li>内容写在这里</li>
			<li>内容写在这里</li>
			...
		</ul>
	</div>
	
当要显示的时候，就改变面板的css样式重新显示出来

样式：

	.hidden{
		display:none;
	}
	
	.shown{
		display:block;
	}
	
那么外观呢？

上面的标签是一个正方形，未激活的时候是灰色边框，激活之后是上面带红色高亮的方框，然后下面面板红色的顶框有跟标签同长度的一段消失掉。

上面的标签好办，我给放置标签的`<ul>`增加类`tabs`，然后增加以下css

	.tabs>li{
		display:inline-block; /*让 li 元素横着排版*/
		margin : 0px 2px; /*设置一个好看的长宽和内边距外边距以及字体样式和位置*/
		padding : 3px 2px;
		width : 70px;
		height : 25px;
		font-size : 13x;
		text-align : center;
		border : 1px solid #ccc; /*未激活的灰色边框*/
		border-bottom : none; /*下边框不要*/
		background-color : white;
	}
	
	/*增加一个伪类 active 表示激活状态*/
	.tabs>li.active{
		border-top : 2px solid #955; /*框顶部的红色*/
		position : relative;
		top : 2px;
		border-bottom : 1px solid #fff;
	}
	
这里有个有趣的地方，因为直接让`<div>`的边框让出跟标签同宽的空白是不可能的，所以想到实际上是标签把下面的`<div>`的边框给遮住了，怎么实现这个效果呢？我想了很久，想到相对定位让`<li>`标签下沉对应的位置就好了。所以就有了上面的`position`和`top`属性的配合使得标签下沉遮住边框。

搞定了样式开始写 Javascript ，原理无非就是给三个`<li>`绑定`onlick`事件，于是直接用`window.onload`来给标签自动添加事件。

	window.onload = function(){
		var tabs = document.getElementById("tabs").getElementByTagName("li");
		for(var i=0;i < tabs.length; i++){
			tabs[i].index = i;
			tabs[i].onclick = function(){
				//标签切换代码
			}
		}
	}

为了方便我给放标签的`<ul>`加了`id`叫`tabs`

这里有个要注意的地方：为什么要用`document.getElementById("tabs").getElementByTagName("li");
`而不是`document.getElementById("tabs").childNodes;`

`childNodes`有浏览器兼容性问题，如果在IE上面用的话，能取到`<ul>`里的`<li>`元素，但是如果在其他浏览器里使用，将不止获取到`<li>`元素，而是连同里面的`TextNode`元素一起获取过来，那么这个列表就不是纯粹的只存放`<li>`的列表了。用`getElementByTagName("li")`能精准拿出所有`<li>`。

还有一行`tabs[i].index = i;`，这个是为了之后的代码能够方便地获取标签的id，因为我们没办法直接获取到发出点击事件的标签在标签栏里的顺序，所以直接给这个标签的节点声明一个`index`会方便很多。

为了方便我给放这个选项卡的容器设置`id="tab-list"`

下面是事件的代码

	tabs[i].onclick = function(){
		var panels = document
			.getElementById("tab-list")
			.getElementsByTagName("div");
			
		for(var i=0;i<panels.length;i++) 
			panels[i].className = "hide";
			
		for(var i=0;i<tabs.length;i++)
			tabs[i].className = "";
			
		this.className = "active";
		panels[this.index].className = "show"; //给标签设置index就是方便在这里直接获取发出事件的标签的顺序。从而给对应的 panel 设置 show 属性。
	}