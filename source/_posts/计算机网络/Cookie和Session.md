#### Cookie机制

- 定义

  Cookie使服务器在本地机器上存储的小段文本并随每一个请求发送至同一服务器

- 产生流程

  正统的cookie分发使通过扩展HTTP协议实现的，服务器通过在HTTP响应头中加一行特殊的指示以提示浏览器按照指示生成相应的cookie，一般这个时候就会把服务器产生的sessionId发送给浏览器保存在cookie中以便维持有状态。当浏览器发送请求时，会检查所有存储的cookie，如果某个cookie所声明的作用范围大于等于将要请求的资源所在位置，则把该cookie附在请求资源的HTTP请求头上发送给浏览器。

#### Session机制

- 定义

  session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构来保存信息

- 产生流程

  当程序需要为某个客户端的请求创建一个session时，服务器首先会检查这个客户端的请求里是否已经包含了一个sessionId，如果有就将该session检索出来放在Request中来使用，如果没有就创建一个新的seesion放在Request来使用，产生的sessionId将被在本次响应中返回给客户端保存，一般保存方式就是cookie

#### 两者的不同点

1. 存取方式的不同

   Cookie只能保存ASCII字符串，存储UNICODE字符需要先进行编码

   Session能保存任何类型的数据甚至java类

2. 隐私策略不同

   Cookie对于客户端是可见的，Session是不可见的

3. 有效期上的不同

   Cookie可以设置一个较长时间的过期时间，但是Session关闭了阅读器该Session就会失效，因而不能长期有效，如果Session长时间有效会导致服务器内存溢出

4. 服务器压力不同 

   Cookie存储在本地，Session存储在服务器

5. 跨域的支持上的不同

   Cookie支持跨域，但是Session不支持跨域，不同域名使用不同Session

#### 关于Cookie和Session的跨域共享

- **正常情况下Cookie发送**

  现在我们有两个站点

  ```
  www.example.com/site1
  www.example.com/site2
  ```

  这两个站点因为在同一个域名下，所以在访问这两个站点时，会发送相同的cookie，因为浏览器存储cookie域是www.example.com

- **我们想实现的是同一个域但是不同的子域如何进行单点登录**

  ```
  sub1.onmpw.com
  sub2.onmpw.com
  ```

  上面就是两个同一个域但是不同的子域的例子，我们可以采用以下的实现方式实现：

  1. 登录sub1.onmpw.com系统 

  2. 登录成功以后，设置cookie信息。这里需要注意，我们可以将用户名和密码存到cookie中，但是在设置的时候必须将这cookie的所属域设置为顶级域 .onmpw.com。这里可以使用setcookie函数，该函数的第四个参数是用来设置cookie所述域的。

     ```
     cookie.setDomain(".onmpw.com");
     ```

  3. 访问sub2.onmpw.com系统，浏览器会将cookie中的信息username和password附带在请求中一块儿发送到sub2.onmpw.com系统。这时该系统会先检查session是否登录，如果没有登录则验证cookie中的username和password从而实现自动登录。

  4. sub2.onmpw.com 登录成功以后再写session信息。以后的验证就用自己的session信息验证就可以了。 

- **但是现在出现了一个问题**

  问题：当我们进入一个子域，要退出时我们可以删除自身的session信息和所属域为.onmpw.com的cookie，但是由于session时不跨域的不能删除另一个子域的session信息，也就是不能同时退出

  解决方案：把第一登录生成的JSESSIONID，通过setDomain放到一个共享的自定义的cookie中。之后访问二级域名的时候，将自定义cookie中的值取出来，然后再放到JESSIONID的cookie值中

  ```
  Cookie c = new Cookie("JSESSIONID", session.getId());  
  c.setDomain("abc.com");  
  resp.addCookie(c);
  ```

  

