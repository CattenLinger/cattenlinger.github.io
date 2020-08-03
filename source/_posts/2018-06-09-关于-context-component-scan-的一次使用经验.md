---
title: '关于 context:component-scan 的一次使用经验'
date: 2018-06-09 06:41:57
categories:
- '笔记'
- '技术'
tags:
- 'Spring'
- 'Spring IoC'
---
问题是这样的，我按照正常的配置文件结构配置，然后每次启动起来都提示找不到 Controller ，反反复复看配置文件没发现问题，依赖也正确，甚至连数据库连接池都换了，也没解决 Controller Not Found 的问题。。。

后来过了一天我想了一下，既然是找不到 Controller ，那么就是 Controller 的类没被找到，那么如果没被找到的话，是不是扫描的时候出了问题呢？

我原来的配置是这样的

```xml
<context:component-scan base-package="cn.lncsa.controller.*"/>
```

然后我试了一下增加了 Controller 的 filter 

```xml
<context:component-scan base-package="cn.lncsa.controller.*">
	<context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

问题并没有被解决。

后来观察，发现 base-package 参数是 "cn.lncsa.controller.\*"，然后回头对比了一下按照教程做的能正常运行的 Helloworld ，是 "net.catten.mvc.\*" 而不是"net.catten.mvc.controller.\*"，我在想是不是这个通配符的问题，于是便试着把 base-package 改成 "cn.lncsa.controller"

```xml
<context:component-scan base-package="cn.lncsa.controller"/>
```

问题解决。。。。

然后在想为什么呢？

是这样的，base-package 指的是扫描器从那个包开始扫描， "cn.lncsa.controller.\*" 指的是扫描 controller 包下面的任何包，也就是说 \* 通配符是改变了扫描基于的包了，不再是 controller 而是 controller 里面的各个子包。而我的 Controller 类是放在这个包下的，扫的是根包里的子包的类而不是根包里的类，当然就搜索不到 Controller 了。

那么这个错误的配置除了上面这么改还能怎么改呢？

```xml
<context:component-scan base-package="cn.lncsa.*">
	<context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

这样改就好了。虽然方法不同但是原理都是更改扫描的根包。第一种方法是直接明确地指出扫描的地方。第二种方法是扫描 cn.lncsa 里面的子包，但是增加 include-filter， 使其只扫描 Controller 类。

当然我的习惯是使用第一种。

还有需要注意的是，因为 SpringMVC 一般是和 Spring 共用， 所以会有重复扫描类的问题。我们的 Controller 是交给 SpringMVC 所属的容器管理的，所以应该让主要的 Spring IOC 容器忽略掉这些 Controller。不然会造成重复扫描，生成重复的对象在两个 IOC 容器里。

所以除了配置 spring-mvc 的 applicationContext 以外，也要在主要的 applicationContxt 里配置扫描器。

```xml
<context:component-scan base-package="cn.lncsa">
	<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

增加 exclude-filter 就能让主要的 IOC 容器忽略掉这些 Controller