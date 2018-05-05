---
title: javaweb中的编码
date: 2018-02-25 23:51:13
tags: javaweb编码
categories: javaweb
---

### JavaWeb中的编码问题

- 编码的原因：
  1. 计算机中存储信息的最小单位是一个字节，所能表示的字符范围是0—255个
  2. 人类要表示的符号太多，无法用1个字节表示。 

#### 常见的编码

1. ASCII
2. ISO-8859-1:256个字符，涵盖大多数西欧字符；
3. GB2312：双字节，中文编码；
4. GBK： GB2312的扩展；
5. GB18030：中文，单双四字节都有；
6. UTF-16：两个字节表示一个字符；
7. UTF-8：运用变长技术，ASCII用单字节表示，中文用三个字节表示。

#### Java中如何解编码

1. 首先会根据指定的charsetName也就是你指定的编码类型，通过Charset.forName(charsetName)找到Charset类，然后根据Charset创建CharsetEncoder对象，再调用CharsetEncoder.encode对字符串编码，不同的编码类型都会对应到一个类中 ，实际的编码过程就是在这些类中完成的。

#### 几种编码方式的比较

对于中文字符由于GBK比GB2312范围更大，编码方式相同，所以GBK更好。UTf-16编码效率更高，只是把一个字节拆成两个字节就完成它的工作，所以在磁盘与内存上的操作它更适合，但是在字节容易损坏的网络上utf-8更适合，它不像utf-16顺序编码，一个字节的损坏会影响后面的字符。且编码效率上utf-8介于GBK和utf-16之间。

#### JavaWeb中涉及的编码

1. url的编解码

   [http://localhost:8080//examples/servlets/serlet/君山?authod=君山](http://localhost:8080//examples/servlets/serlet/%E5%90%9B%E5%B1%B1?authod=%E5%90%9B%E5%B1%B1)

   sheme domin port contextPath servletPath pathInfo queryString

   一般情况下pathInfo采用的是utf-8编码，然后浏览器会将非ASCII字符按照某种编码格式编码成16进制数字后将每个16进制数字表示的字节前加上%，Tomcat的设置上默认也是按照utf-8解码pathInfo；queryString是按照传输中在header中设置的contentType编码方式进行编码，然后在Tomcat中可以设置useBodyEncodingForURI=“true”将queryString的解码方式采用的也是contentType设置的编码方式。

2. HTTP Header 的编解码

   header中只能传输ASCII字符，不能采用其他编解码方式；如果一定要传递可以先将这些字符用URIEncoder编码再添加到Header中。

3. Post表单上的编解码

   在POST提交的时候会按照ContentType编码，tomcat也会按照ContentType解码；Tomcat在getParameter方法之前会获取contentType中的charset；但是这里有一种特殊情况，在 POST提交表单的时候默认情况下 ContentType是不会有值的就会采用默认的ISO-8859-1；而在jsp中设置的contentType中的Charset是告诉浏览器该页面该用什么解码。

4. HTTP body的编解码

   在服务器通过response返给浏览器的时候与request给服务器的编解码过程差不多；通过设置response的contentType然后服务器和浏览器就用这个charset对body进行编解码。