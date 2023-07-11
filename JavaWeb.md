# Servlet

## Servlet生命周期
<img src="https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/3252049_1503478423363_D9B23E5A78E8071F0BB0D7E0E456058E" alt="img" style="zoom:200%;" />

## Servlet结构
<img src="https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/9430388_1508834386231_FE8B1A979ADF6E3C2C114AF3F9CA693C" alt="img" style="zoom:150%;" />



## Servlet参数
---

**HttpSessionAttributeListener**：可以实现此侦听器接口获取此web应用程序中会话属性列表更改的通知；
**HttpSessionBindingListener**：当该对象从一个会话中被绑定或者解绑时通知该对象，这个对象由HttpSessionBindingEvent对象通知。这可能是servlet程序显式地从会话中解绑定属性的结果，可能是由于会话无效，也可能是由于会话超时；
**HttpSessionObjectListener**：没有该接口API
**HttpSessionListener**：当web应用程序中的活动会话列表发生更改时通知该接口的实现类，为了接收该通知事件，必须在web应用程序的部署描述符中配置实现类；
**HttpSessionActivationListener**：绑定到会话的对象可以侦听容器事件，通知它们会话将被钝化，会话将被激活。需要一个在虚拟机之间迁移会话或持久会话的容器来通知所有绑定到实现该接口会话的属性。

下面的方法可用在Servlet程序中读取Http头 这些方法通过HttpServletRequest对象可用:

- Cookie[] getCookies()     返回一个数组,包含客户端发送该请求的所有Cookie对象
- Object getAttribute(String name)   以对象形式返回已命名的属性的值,如果没有给定名称的属性存在,则返回null
- String getHeader(String name)  以字符串形式返回指定的请求头的值  Cookie也是头的一种
- String getParameter(String name)  以字符串形式返回请求参数的值,或者如果参数不存在则返回null

---
## TCP三次握手过程和四次挥手过程
---
### 三次握手(连接建立)
    所谓三次握手(Three-way Handshake)，是指建立一个 TCP 连接时，需要客户端和服务器总共发送3个包。
    
    三次握手的目的是连接服务器指定端口，建立 TCP 连接，并同步连接双方的序列号和确认号，交换 TCP 窗口大小信息。
    在 socket 编程中，客户端执行 connect() 时。将触发三次握手。

- 第一次握手(SYN=1, seq=x):

    客户端发送一个 TCP 的 SYN 标志位置1的包，指明客户端打算连接的服务器的端口，以及初始序号 X,保存在包头的序列号(Sequence Number)字段里。

    发送完毕后，客户端进入 SYN_SEND 状态。

- 第二次握手(SYN=1, ACK=1, seq=y, ACKnum=x+1):

    服务器发回确认包(ACK)应答。即 SYN 标志位和 ACK 标志位均为1。服务器端选择自己 ISN 序列号，放到 Seq 域里，同时将确认序号(Acknowledgement Number)设置为客户的 ISN 加1，即X+1。 发送完毕后，服务器端进入 SYN_RCVD 状态。

- 第三次握手(ACK=1，ACKnum=y+1)

    客户端再次发送确认包(ACK)，SYN 标志位为0，ACK 标志位为1，并且把服务器发来 ACK 的序号字段+1，放在确定字段中发送给对方，并且在数据段放写ISN的+1

    发送完毕后，客户端进入 ESTABLISHED 状态，当服务器端接收到这个包时，也进入 ESTABLISHED 状态，TCP 握手结束。
    

### 四次挥手(释放连接)
---
- 第一次挥手(FIN=1，seq=x)

    假设客户端想要关闭连接，客户端发送一个 FIN 标志位置为1的包，表示自己已经没有数据可以发送了，但是仍然可以接受数据。

    发送完毕后，客户端进入 FIN_WAIT_1 状态。

- 第二次挥手(ACK=1，ACKnum=x+1)

    服务器端确认客户端的 FIN 包，发送一个确认包，表明自己接受到了客户端关闭连接的请求，但还没有准备好关闭连接。

    发送完毕后，服务器端进入 CLOSE_WAIT 状态，客户端接收到这个确认包之后，进入 FIN_WAIT_2 状态，等待服务器端关闭连接。

- 第三次挥手(FIN=1，seq=y)

    服务器端准备好关闭连接时，向客户端发送结束连接请求，FIN 置为1。

    发送完毕后，服务器端进入 LAST_ACK 状态，等待来自客户端的最后一个ACK。

- 第四次挥手(ACK=1，ACKnum=y+1)

    客户端接收到来自服务器端的关闭请求，发送一个确认包，并进入 TIME_WAIT状态，等待可能出现的要求重传的 ACK 包。

    服务器端接收到这个确认包之后，关闭连接，进入 CLOSED 状态。

    客户端等待了某个固定时间（两个最大段生命周期，2MSL，2 Maximum Segment Lifetime）之后，没有收到服务器端的 ACK ，认为服务器端已经正常关闭连接，于是自己也关闭连接，进入 CLOSED 状态

---
## HTTP和HTTPS的区别
---
1、https协议需要到ca申请证书，一般免费证书较少，因而需要一定费用。
2、http是超文本传输协议，信息是明文传输，https则是具有安全性的ssl加密传输协议。
3、http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。
4、http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全


## TCP和UDP的区别

---
1. TCP是面向连接的,UDPUDPUDP是无连接的
2. TCP是可靠传输,使用了流量控制和拥塞控制,UDPUDP是不可靠传输
3. TCP只能支持一对一通信,UDP支持一对一/一对多/多对一/多对多交互通信
4. TCP的传输面向字节流,UDP的传输面向报文
5. xxxxxxxxxx  public static void main(String[] args) {        int[] demo = new int[] { -2, -435, -12, -432, 43, 4 };        int end = demo.length - 1;        int sum = 0;        System.out.println(Arrays.toString(demo));        quickSort(demo, 0, end);        System.out.println(Arrays.toString(demo));        if (demo[end] < 0) {            System.out.println("0");            return;        }        int res = 0;        for (int i = end; i >= 0; --i) {            sum += demo[i];            if (sum >= 0) {                res++;            }                    }        System.out.println(res);    }    // 快排算法​    private static void quickSort(int[] demo, int head, int end) {        // TODO Auto-generated method stub        if (head >= end)            return;        int low = head;// 头指针        int high = end;// 尾指针        int pivot = demo[low]; // 基准        while (low < high) {            while (low < high && demo[high] > pivot) {                high--;            }            while (low < high && demo[low] <= pivot) {                low++;            }            // 找到交换            swap(demo, low, high);        }        // 没找到交换        swap(demo, low, head);        quickSort(demo, head, low - 1);        quickSort(demo, low + 1, end);    }​    private static void swap(int[] demo, int low, int high) {        int temp = demo[low];        demo[low] = demo[high];        demo[high] = temp;    }java
6. TCP适用于要求可靠传输的应用,UDP适用于实时应用

## 转发的重定向的区别
---
>请求转发：
>
>```java
>request.getRequestDispatcher("/test.jsp").forword(request,response);
>```
>
>重定向：
>
>```java
>response.sendRedirect("/test.jsp");
>```

| 区别点       | 转发                                                         | 重定向                                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 请求次数     | 服务器收到请求后<br />为了完成响应跳转到一个新的地址(**请求一次**) | 浏览器向服务器发送一个请求**并收到响应后**<br />再次向一个新地址发出请求(**至少请求两次**) |
| 浏览器地址栏 | 地址栏不会发生变化                                           | 地址栏会发生变化                                             |
| 是否共享数据 | 数据共享(一次请求)                                           | 数据不共享(多次请求)                                         |
| 跳转限制     | 只能跳转本站点资源                                           | 可以跳转到任意URL                                            |
| 发生行为不同 | 服务端行为                                                   | 客户端行为                                                   |

## Cookie和Session的区别
---
- 数据存放位置不同：

    cookie数据存放在客户的浏览器上，session数据放在服务器上

- 安全程度不同：

    cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗,考虑到安全应当使用session。

- 性能使用程度不同：

    session会在一定时间内（默认30分钟）保存在服务器上。当访问增多，会比较占用你服务器的性能,考虑到减轻服务器性能方面，应当使用cookie。

- 数据存储大小不同：

    单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie，而session则存储与服务端，浏览器对其没有限制。
---
###什么是SQL注入?如何避免SQL注入?

- 什么是SQL注入

>		SQL注入即是指web应用程序对用户输入数据的合法性没有判断或过滤不严，攻击者可以在web应用程序中事先定义好的查询语句的结尾上添加额外的SQL语句，在管理员不知情的情况下实现非法操作，以此来实现欺骗数据库服务器执行非授权的任意查询，从而进一步得到相应的数据信息。

- 如何避免SQL注入

  >1. 严格限制Web应用的数据库的操作权限，给此用户提供仅仅能够满足其工作的最低权限，从而最大限度的减少注入攻击对数据库的危害。
  >2. 检查输入的数据是否具有所期望的数据格式，严格限制变量的类型，例如使用regexp包进行一些匹配处理，或者使用strconv包对字符串转化成其他基本类型的数据进行判断。
  >3. 对进入数据库的特殊字符（'"\尖括号&*;等）进行转义处理，或编码转换
  >4. 所有的查询语句建议使用数据库提供的参数化查询接口，参数化的语句使用参数而不是将用户输入变量嵌入到SQL语句中，即不要直接拼接SQL语句。例如使用`database/sql`里面的查询函数`Prepare`和`Query`，或者`Exec(query string, args ...interface{})`。
  >5. 在应用发布之前建议使用专业的SQL注入检测工具进行检测，以及时修补被发现的SQL注入漏洞。网上有很多这方面的开源工具，例如sqlmap、SQLninja等。
  >6. 避免网站打印出SQL错误信息，比如类型错误、字段不匹配等，把代码里的SQL语句暴露出来，以防止攻击者利用这些错误信息进行SQL注入。
---

## 什么是跨域请求? 如何解决跨域请求问题
---
什么是跨域请求

>当前发起请求的域与该请求指向的资源所在的域不一样

1. 跨域解决方法

   - JSONP

     >​		JSONP 是一种非官方的跨域数据交互协议
     >
     >​		JSONP 本质上是利用 `<script><img><iframe>` 等标签不受同源策略限制，可以从不同域加载并执行资源的特性，来实现数据跨域传输。
     >
     >​		JSONP由两部分组成：回调函数和数据。回调函数是当响应到来时应该在页面中调用的函数，而数据就是传入回调函数中的JSON数据。
     >
     >​		JSONP 的理念就是，与服务端约定好一个回调函数名，服务端接收到请求后，将返回一段 Javascript，在这段  Javascript 代码中调用了约定好的回调函数，并且将数据作为参数进行传递。当网页接收到这段 Javascript 代码后，就会执行这个回调函数，这时数据已经成功传输到客户端了

   - CORS

     >​		跨源资源共享 **Cross-Origin Resource Sharing(CORS)** 是一个新的 W3C 标准，它新增的一组HTTP首部字段，允许服务端其声明哪些源站有权限访问哪些资源。换言之，它允许浏览器向声明了 CORS 的跨域服务器，发出 XMLHttpReuest 请求，从而克服 Ajax 只能同源使用的限制。
---
