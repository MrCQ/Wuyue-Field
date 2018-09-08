# Tomcat+Servlet+SpringMVC的关系简析

## Servlet 是什么

Servlet并不是一个多么高深的框架组件，它仅仅只是一个标准规范，确定了Java处理网络请求的规范。具体看可以理解为它定义了一个接口，Java实现网络请求处理都需要实现Servlet中定义的接口，接口中有5个方法，主要有：

* init() : 初始化阶段的处理
* destroy() : 销毁阶段的处理
* service() : 接受到网络请求时的处理

Servlet只是一个规范，它定义了Java中网络请求处理的基本逻辑，每个Servlet的实现会对应到一个网络请求，但是Servlet实例自身仅仅是对于网络请求的处理，它自身并不能完成与客户端的网络连接等操作，这就需要用到Servlet容器

## Servlet 容器是什么

首先，Tomcat就是典型的Servlet容器，Servlet实例自身仅仅只能完成对于网络请求的上层处理，并不能做到如：

* 监听特定端口的连接请求
* 与客户端完成通信
* 获取资源

这些Servlet实例自身都不能完成，谁来完成呢？ Servlet容器来完成，也就是Tomcat帮助完成这些工作，容器会监控特定端口的连接，解析连接请求并映射到指定的Servlet实例上，交给Servlet实例来完成处理，拿到处理结果后再由容器向客户端返回。


## Servlet in Tomcat 

Tomcat接口到网络请求后，Servlet的处理流程如下：

1. Tomcat从其Connector中接口网络请求
2. Tomcat会将网路请求映射到指定的Engine
3. 一旦请求被映射到对应的Servlet，Tomcat会检查对应的Servlet类是否被加载，如果没有则加载类并创建Servlet实例
4. Tomcat会通过调用Servlet类的init()方法初始化Servlet实例，初始化过程会读取Tomcat的相关配置
5. Servlet实例初始化完成后，Tomcat会调用其service()方法来处理网络请求，该方法会返回Response
6. 在Servlet实例的整个生命周期中，Tomcat与Servlet实例之间能够通过Listener来交互，Listener会监控Servlet的状态
7. 当Tomcat关闭时，Tomcat会调用Servlet实例的destroy()方法完成销毁

## SpringMVC

SpringMVC 是 Spring 框架最重要的模块之一，是实现了Web MVC设计模式的请求驱动类型的轻量级Web框架

一个典型的代码案例：

```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.servlet.ModelAndView;

@PostMapping("/login")
public ModelAndView login(LoginData loginData) {
    if (LOGIN.equals(loginData.getLogin()) 
      && PASSWORD.equals(loginData.getPassword())) {
        return new ModelAndView("success", 
          Collections.singletonMap("login", loginData.getLogin()));
    } else {
        return new ModelAndView("failure", 
          Collections.singletonMap("login", loginData.getLogin()));
    }
}
```

SpringMVC依托Tomcat等Servlet容器来运行，HTTP请求自然都是由Tomcat应用服务器来处理，再由其Servlet容器管理Servlet的执行，那么对于SpringMVC的Web应用而言，其执行入口必然就是Servlet实例。

SpringMVC的关键就在于**DispatcherServlet**，它就是SpringMVC的Servlet入口，任何指向该SpringWeb应用的HTTP请求都会被Tomcat映射到该Servlet，由DispatcherServlet作进一步解析、映射、处理与结果渲染，最后返还给Tomcat的Servlet容器，容器再交给Tomcat的Web组件完成网络回传。

SpringMVC的处理过程紧紧围绕DispatcherServlet展开，其工作原理如图所示：

![](http://ww4.sinaimg.cn/mw690/6941baebtw1epg9al8bv6j20f90aqjrx.jpg)

参见：

* [An Introduction to Tomcat Servlet Interactions](https://www.mulesoft.com/cn/tcat/tomcat-servlet)
* [How Spring Web MVC Really Works](https://stackify.com/spring-mvc/)