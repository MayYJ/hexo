---
title: 关于Tomcat
date: 2018-10-29 10:09:06
tags: tomcat
categories: javaweb
---

#### 概述

​虽然以前通过SpringBoot的源码看了一些其内置的Tomcat源码，但是因为看得比较马虎   :disappointed:  所以只知道大概，今天要根据自己看得几篇博客和Tomcat源码来总结下在我看来的Tomcat，然后后面准备自己模仿Tomcat实现一个自己的HTTP服务器，想想还是很激动 :laughing:

####  Tomcat

Tomcat 服务器是一个免费的开放源代码的Web 应用服务器，属于轻量级应用[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调试JSP 程序的首选。 

#### Tomcat顶层架构

![](https://img-blog.csdn.net/20180109094904328?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzgyNDU1Mzc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](https://img-blog.csdn.net/20180109095032618?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzgyNDU1Mzc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 大致关系

一个Server可以有多个Service

一个Service只能有一个Container

一个Service拥有多个Connector

一个Container可以有多个Context

##### 组件功能

**Server**

拥有自己的生命周期，在这里讲解一下声明是声明周期，首先它可以控制组件整个流程，然后就是能够监听生命周期中的事件，方便用户自定义；主要控制Tomcat的生死大权

**Service**

顾名思义服务，我们根据Service接口，差不多也知道了它的主要功能

```java
public interface Service extends Lifecycle {
    Engine getContainer();

    void setContainer(Engine var1);

    String getName();

    void setName(String var1);

    Server getServer();

    void setServer(Server var1);

    ClassLoader getParentClassLoader();

    void setParentClassLoader(ClassLoader var1);

    String getDomain();

    void addConnector(Connector var1);

    Connector[] findConnectors();

    void removeConnector(Connector var1);

// 这里会添加到Executor链表中，并且会在线程池中取运行
    void addExecutor(Executor var1);

    Executor[] findExecutors();

    Executor getExecutor(String var1);

    void removeExecutor(Executor var1);

    Mapper getMapper();
}

```

1. 取得子组件
2. 取得父组件
3. 管理Connectors
4. 将任务丢进线程池里面去运

**Engine**

我们依然使用接口来了解它的功能

```
public interface Engine extends Container {
    String getDefaultHost();

    void setDefaultHost(String var1);

    String getJvmRoute();

    void setJvmRoute(String var1);

    Service getService();

    void setService(Service var1);
}
```

1. 与Service建立上下级关系
2. 管理Host,即管理站点，下面我们详细介绍这里的Host即站点是什么

**Host**

站点，虚拟主机，使用的情况是在同一台服务器下部署两个应用，但是想用不同域名访问这两个应用就可以使用不用的Host, 主要功能就是管理Context，即封装和管理Servlet

![](https://img-blog.csdn.net/20180109110049444?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzgyNDU1Mzc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上面这张图是对Engine和Host的总结，这里不知道有没有一个为什么要把Servlet封装成Wrapper的疑问，因为Container接口里面findChild 返回类型为Container，我们必须将Servlet封装成Wrapper，刚好也把所有ServletContext中设置的参数放进这个wapper中

**Connector**

![](https://img-blog.csdn.net/20180109095336437?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzgyNDU1Mzc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

主要功能就是用于接受请求并将请求封装成Request和Response来具体处理；  其中具体处理过程如上图

Endpoint：处理底层Socket的网络连接 

Processor : Endpoint接收到的Socket封装成Request 

Adapter: 将Request交给Container进行具体的处理 

#### 参考

https://blog.csdn.net/qq_38245537/article/details/79009448

