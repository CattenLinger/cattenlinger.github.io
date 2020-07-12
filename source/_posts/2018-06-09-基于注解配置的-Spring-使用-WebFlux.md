---
title: 基于注解配置的 Spring 使用 WebFlux
date: 2018-06-09 06:51:47
categories:
- '笔记'
tags:
- 'Java'
- 'Spring'
- 'WebFlux'
---
我有一个 Netty 的项目要增加一个 HTTP Server ，不能改成 Spring Boot ，但是本身使用 Spring 作依赖注入，然后就想着能不能直接使用 Netty 处理这些 Http 请求。。。在网上搜了半天，感觉往里面塞个 Servlet 不太好，于是还是回去用 Spring 的 WebFlux。虽然 WebFlux 一般都捆着 Spring Boot ，但是想着应该可以单独使用吧。。。于是就试了一下。。

一开始先找到这里 [23.3.2 Manual Bootstrapping](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.html#web-reactive-getting-started-manual) ，结果半天找不到 `DispatcherHandler.toHttpHandler()` 方法（懵）然后就想了些歪门邪道了（x



Maven 依赖

```xml
<properties>
  <spring.version>5.0.4.RELEASE</spring.version>
  <jackson.version>2.9.0</jackson.version>
</properties>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>${jackson.version}</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>${jackson.version}</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-annotations</artifactId>
  <version>${jackson.version}</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>${spring.version}</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context-support</artifactId>
  <version>${spring.version}</version>
</dependency>

<!-- Reactor + Webflux 依赖 -->

<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.13.Final</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webflux</artifactId>
  <version>${spring.version}</version>
</dependency>

<dependency>
  <groupId>io.projectreactor.ipc</groupId>
  <artifactId>reactor-netty</artifactId>
  <version>0.7.5.RELEASE</version>
</dependency>

<dependency>
  <groupId>io.projectreactor</groupId>
  <artifactId>reactor-core</artifactId>
  <version>3.1.5.RELEASE</version>
</dependency>
```

`ApplicationConfigure.java` 不需要写什么东西

```java
@Configuration
@EnableWebFlux
@ComponentScan("com.example")
public class ApplicationConfigure {
  
}
```

`ApiServer.java`	 是我项目里的 HTTP 服务器

```java
@Component
public class ApiServer{

    private ApplicationContext applicationContext;
    private HttpServer httpServer;

    public ApiServer(ApplicationContext applicationContext){
        this.applicationContext = applicationContext;
        httpServer = HttpServer.create("0.0.0.0",8080);
    }

  	// 可以丢到 Configuration 里面，这样就不需要像我这样注册钩子了
    @PostConstruct
    public void start(){
        httpServer.start(new ReactorHttpHandlerAdapter(
                WebHttpHandlerBuilder
                        .applicationContext(applicationContext)
                        .build()
        ));
    }

  	// 习惯性写上去的没啥用。。。
    @PreDestroy
    public void stop(){
    }

}
```

随便写个 `HelloController.java` 。这里类注解要用 `@RestController` ，不然返回结果的时候会冒 **viewResolver Not Found** 之类的错误。

```java
@RestController
public class HelloController {
    @GetMapping
    public String hello() {
        return "Hello world";
    }
}
```

启动用的 `Main.java`

```java
public class Main {
    private final static Logger logger = LoggerFactory.getLogger(Loggers.APPLICATION);

    public static void main(String... args) {
        logger.debug("Loading application context...");

        AbstractApplicationContext context = new AnnotationConfigApplicationContext(ApplicationConfig.class);
      	// 生命周期钩子用的而已
        context.registerShutdownHook();

        logger.info("Application context loaded.");
    }
}
```

然后访问 `http://localhost:8080` 就出来一个 HelloWorld 囖~

关于视图或者其他的话还得去查一下 WebFlux 相关的内容。