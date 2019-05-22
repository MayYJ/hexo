---
title: Nginx配置记录
date: 2019-03-21 11:28:30
tags: Nginx
category : javaweb
---

#### Location

location块用于定位相应URL的处理

Nginx对location块有三种处理方式，处理的步骤也是按照下面的步骤

1.  完全匹配，使用"="

   ```java
   location = / {
       [ configuration A ]
   }
   ```

2. 最长前缀匹配

   ```
   location / {
       [ configuration B ]
   }
   
   location /documents/ {
       [ configuration C ]
   }
   ```

3. 正则表达式匹配

   ```
   location \.(gif|jpg|jpeg)$ {
       [ configuration E ]
   }
   ```

- 还有其它注意的点

  1. 使用"^~" 大小写敏感的匹配URL

  ```
  location ^~ /images/ {
      [ configuration D ]
  }
  ```

  2. 使用"~*" 大小写不敏感的匹配URL

     ```
     location ~* \.(gif|jpg|jpeg)$ {
         [ configuration E ]
     }
     ```

  3. 对于非"/"结尾的URL会被重定位到相应的以"/"结尾的URL上处理，所以如果两者有差别就分别进行处理

     ```
     location /user/ {
         proxy_pass http://user.example.com;
     }
     
     location = /user {
         proxy_pass http://login.example.com;
     }
     ```

     