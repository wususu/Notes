# 浅析Web容器

## 容器介绍

容器这个概念是针对Servlet而言的,Servlet一般指的是Java Servlet<br/>
Web容器用于管理Servlet，当服务器接收到一个指向Servlet的请求，就将其转发给Web容器<br/>
之后由Web容器向Servlet提供http请求(request),http响应(response),再通过调用service()方法(doPost或doGet)进行处理

## 容器的作用

1. 提供通讯支持:

利用容器提供的方法，可以简单的实现servlet与web服务器的对话。
否则就要自己建立server，监听端口，创建新的流等等一系列复杂的操作。(就是直接Socket编程了)而容器的存在就帮我们封装这一系列复杂的操作。使我们能够专注于servlet中的业务逻辑的实现。

2. 生命周期管理
容器负责servlet的整个生命周期。如何加载类，实例化和初始化servlet，调用servlet方法，并使servlet实例能够被垃圾回收。有了容器，我们就不用花精力去考虑这些资源管理垃圾回收之类的事情。

3. 多线程支持
容器会自动为接收的每个servlet请求创建一个新的java线程，servlet运行完之后，容器会自动结束这个线程。

4. 声明式安全
利用容器，可以使用xml部署描述文件来配置安全性，而不必将其硬编码到servlet中

5. jsp支持
容器可以jsp翻译成java文件(本质是Servlet)

## 容器处理请求流程

1. client请求一个指向Servlet的url
2. 容器识别出请求索要的是Servlet,新建出`httpServletRequest`和`httpServletResponse`两个对象
3. 容器根据请求的url和web.xml配置文件的映射关系找到对应的Servlet,为当前请求新建一个线程,并把`httpServletRequest`,`httpServletResponse`传递到Servlet对象中
4. Servlet对象通过调用`service()`方法调用`doGet()`或`doPost()`方法处理请求
5. 请求处理完成后将响应内容填入`httpServletResponse`对象中
6. 线程结束,容器将`httpServletResponse`对象转化为http响应,传回给client,销毁`httpServletRequest`和`httpServletResponse`对象


> 内容引自[web开发中 web 容器的作用（如tomcat）-六尺帐篷](https://www.jianshu.com/p/99f34a91aefe)