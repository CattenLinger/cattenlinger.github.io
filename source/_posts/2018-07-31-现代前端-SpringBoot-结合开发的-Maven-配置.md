---
title: 现代前端 + SpringBoot 结合开发的 Maven 配置
date: 2018-07-31 07:06:39
categories:
- '笔记'
tags:
- 'Maven'
- 'Web App'
---
好些日子之前就在网上看见一篇文章，说一个小后端想用 Vue 作前端技术结合 SpringBoot 后端作开发，但又想方便点让前端的工程能够自己跑进 Jar 包里。很感兴趣诶，于是就动手跟着实现一遍。

原文：[A Lovely Spring View: Spring Boot & Vue.js](https://blog.codecentric.de/en/2018/04/spring-boot-vuejs/)

原理实际是利用 frontend-maven-plugin 来调用 node ，不过这个插件有个好处，它是在工程的目录下安装 node，这样能摆脱对本机 node 的依赖，在很多地方进行构建。

起来先建一个普通的 SpringBoot 工程项目，然后普通地把 `src/` 删掉开始建立子模块。我是建立了 backend 和 frontend 两个模块。

```
springboot-vue/
	|- frontend/
	|	|- src/
	|	|	|- ...
	|	|- pom.xml
	|- backend/
	|	|- src/
	|	|	|- ...
	|	|- pom.xml
	|- pom.xml
```

根 pom ：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>net.catten</groupId>
    <artifactId>springboot-vue</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <modules>
        <module>backend</module>
        <module>frontend</module>
    </modules>
    <packaging>pom</packaging>

    <name>project-management</name>
    <description>Team project progress management system</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
    </dependencies>
</project>
```

后端，backend 需要配置一下 `maven-resource-plugin` 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springboot-vue</artifactId>
        <groupId>net.catten</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>backend</artifactId>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <artifactId>maven-resources-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy Vue frontend content</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>src/main/resources/static</outputDirectory>
                            <overwrite>true</overwrite>
                            <resources>
                                <resource>
	                           		<directory>
    	                                ${project.parent.basedir}/frontend/dist
                                    </directory>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

__对这个插件熟悉的就可以跳过下面说明了。__

其中 plugin 的 configuration 内，outputDirectory 是指输出的地方， resource 指从哪里复制文件，复制的文件当然就是 webpack 打包出来的了。

前端，配置 `frontend-maven-plugin`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springboot-vue</artifactId>
        <groupId>net.catten</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>frontend</artifactId>

    <build>
        <plugins>
            <plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <version>1.6</version>
                <executions>
                    <!-- Install our node and npm version to run npm/node scripts-->
                    <execution>
                        <id>install node and npm</id>
                        <goals>
                            <goal>install-node-and-npm</goal>
                        </goals>
                        <configuration>
                            <nodeVersion>v9.11.1</nodeVersion>
                        </configuration>
                    </execution>
                    <!-- Install all project dependencies -->
                    <execution>
                        <id>npm install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <!-- optional: default phase is "generate-resources" -->
                        <phase>generate-resources</phase>
                        <!-- Optional configuration which provides for running any npm command -->
                        <configuration>
                            <arguments>install</arguments>
                        </configuration>
                    </execution>
                    <!-- Build and minify static files -->
                    <execution>
                        <id>npm run build</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <configuration>
                            <arguments>run build</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

这个基本上不用去动，如果对 nodeVersion 有要求可以修改一下。这里的配置是会在 generate-resources 的时候调用 npm run build ，实际拷贝资源文件的操作还是 backend 内的插件做的，所以拷贝的源文件路径需要和 webpack 的配置配合，按需修改 backend 的 configurations。

~~是一篇只知道怎么用不去深究喂到嘴边式快餐文（逃）~~
