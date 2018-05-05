---
title: 关于spring-data
date: 2018-03-09 11:51:06
tags: spring-date redis
categories: javaweb
---

1. 只导入spring-date-redis是会在运行的时候报错的，需要自己导入下面两个包

   ```
           <dependency>
               <groupId>org.apache.commons</groupId>
               <artifactId>commons-pool2</artifactId>
               <version>2.0</version>
           </dependency>
           
           <dependency>
               <groupId>redis.clients</groupId>
               <artifactId>jedis</artifactId>
               <version>2.9.0</version>
           </dependency>
   ```

2. spring-data-redis 2.x 只适合spring5.x hespring boot2.x