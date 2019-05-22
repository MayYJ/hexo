---
title: 一步一步理解dubbo
date: 2019-03-03 22:13:13
tags: dubbo
---

#### JAVA SPI

SPI 全称为 (Service Provider Interface)，是JDK内置的一种服务发现机制；SPI是一种动态替换发现的机制， 比如有个接口，想运行时动态的给它添加实现，你只需要添加一个实现。JAVA 中主要用java.util.ServiceLoader 类实现服务发现

**JDBC服务发现机制**

在JDBC4.0 之前 我们需要 Class.forName("com.mysql.jdbc.Driver") 这样一条代码来主动加载Mysql的驱动类；但是在JDBC4.0之后，我们可以不用写这样的代码，JDBC内部提供了服务发现机制

下面是DriverManager 的静态代码块

```java
static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```

下面是 loadInitialDrivers()的实现

```java
private static void loadInitialDrivers() {
        String drivers;
        ...
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });
        
        ....
        
        }
    }

```

1. 实例化生成对应要加载接口对应实现的 ServiceLoader
2. iterator 会搜索classpath下以及jar包中所有的META-INF/services 目录下java.sql.Driver文件，并找到文件中的实现类的名字

 