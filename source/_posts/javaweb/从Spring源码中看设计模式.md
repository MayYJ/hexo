---
title: 从Spring源码中看设计模式
date: 2019-01-14 10:30:16
tags: Spring
category: javaweb
---

#### 适配器模式

1. HandlerAdapter

   在Spring MVC中存在多个Controller，而每个Controller处理的方式不同，如果不按照适配器模式去做我们正常的处理方式会像下面这样：

   ```java
   if(a instanceof B) {
       B b = (B)a；
       b.doSomethingB();
   }else if(a instanceof C) {
       C c = (C)a;
       c.doSomethingC();
   }....
   ```

   如果这样的话，要添加一个新的Controller，就只能在这个分支选择里面再添加一条，那么就会违背开放闭合原则

   Spring 使用适配器模式 将判断类型和在该类型下面的动作全部分离了出来

   ```java
   public interface HandlerAdapter {
       boolean supports(Object var1);
   
       @Nullable
       ModelAndView handle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;
   
       long getLastModified(HttpServletRequest var1, Object var2);
   }
   ```

   那么在添加一个新类型的时候，只需要创建一个该类型的Adapter，然后再将该Adapter添加到Adapter列表里面

   执行的时候取得Controller 处理后返回的类型，然后遍历Adapter列表找到属于它的Adapter，再直接通过Adapter调用处理方法

2. AdvisorAdapter

   在Spring AOP中需要将所有Advisor 转换成MethodInterceptor形成一个线性表，然后执行方法的时候就遍历执行这个表就行

   这里的Advisor 有不同类型且对应不同的MethodInterceptor

   如果使用分支选择就会下面这样：

   ```java
   if(a instanceof B) {
       B b = new B();
       return b;
   }else if(a instanceof C) {
       C c = new C();
       return c;
   }....
   ```

   但是这样就不具有扩展性了，所以也跟上面一样 判断类型 和具体的处理进行了分离

   ```java
   public interface AdvisorAdapter {
    
   	boolean supportsAdvice(Advice advice);
   	
   	MethodInterceptor getInterceptor(Advisor advisor);
    
   }
   ```

#### 责任链模式

1. Filter链
2. HandlerExecutionChain
3. Aop中的MethodInterceptor

#### 观察者模式

1. Tomcat生命周期Listener
2. Spring事件监听器

#### 工厂方法模式

1. 不同WebServer的构建