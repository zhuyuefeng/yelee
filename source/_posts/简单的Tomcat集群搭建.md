title: 简单的Tomcat集群搭建
date: '2019-05-07 16:05:32'
tags: [Tomcat, Apache, 集群]
categories: [课余笔记]
---
&emsp;&emsp;之前也实现过简单的Tomcat集群，今天突然被问起竟然有些记不清了，还是记下来才是王道😁😁。

<!-- more -->

## 前期准备

+ Windows环境
+ Apache2.4
+ tomcat7(多个)
+ mod_jk_2.4(版本一定要与apache一致)

## 软件安装与配置

### Apache安装

#### 安装命令

&emsp;&emsp;（Apache的下载这里就不赘述了，英文看不懂百度上也有下载教程。）
&emsp;&emsp;因为我是在Windows环境下操作的，所以需要以**管理员的身份**运行cmd，进入到下载并解压好的    `Apache24/bin` 目录下，命令如下：

```
D:\httpd-2.4.39-o102r-x64-vc14\Apache24\bin>httpd.exe -k install
```

#### 启动命令

&emsp;&emsp;安装完成后，需要启动Apache服务，不过在此之前，需要对配置文件做相应的修改：修改 `Apache24\conf` 目录下`httpd.conf`文件的第**38**行如下：

```conf
Define SRVROOT "D:\httpd-2.4.37-o102p-x64-vc14\Apache24" //为当前Apache24的安装路径
ServerRoot "${SRVROOT}"
```
&emsp;&emsp;启动命令如下：

```
D:\httpd-2.4.39-o102r-x64-vc14\Apache24\bin>httpd.exe -k start
```

&emsp;&emsp;停止命令如下：

```
D:\httpd-2.4.39-o102r-x64-vc14\Apache24\bin>httpd.exe -k stop
```

#### 常见报错

+ > “(OS 5)拒绝访问。 : AH00369: Failed to open the Windows service manager..."

&emsp;&emsp;错误原因：没有用管理员身份运行cmd

+ > httpd: Syntax error on line 532 of D:/work/Apache24/conf/httpd.conf: Syntax error on line 3 of
D:/work/Apache24/conf/mod_jk.conf: Cannot load modules/mod_jk-1.2.31-httpd-2.2.3.so into server: %1\xb2\xbb\xca\xc7\xd3\xd0\xd0\xa7\xb5\xc4 Win32 \xd3\xa6\xd3\xc3\xb3\xcc\xd0\xf2\xa1\xa3

&emsp;&emsp;错误原因：apache和mod_jk的版本不一致

+ >443端口占用

&emsp;&emsp;修改方法：找到`Apache24\conf\extra` 下面的`httpd-ahssl.conf`和`httpd-ssl.conf`，有443的地方改为442就可以了。

+ >其他错误可见百度，我在部署时没有遇到其他情况

### mod_jk 连接模块 

#### mod_jk.so下载地址

&emsp;&emsp;[点击进入下载界面](http://archive.apache.org/dist/tomcat/tomcat-connectors/jk/binaries/windows/) （嘿嘿，讲真的我自己也不能确定Apache对应的mod_jk版本是啥，试了几次才找对😁，惭愧惭愧。）

#### 操作步骤
&emsp;&emsp;1、把`mod_jk.so`文件拷贝到`Apache24\modules`下
&emsp;&emsp;2、在`Apache24\conf`目录下新建`mod_jk.conf`、`workers.properties`文件（为了保持`httpd.conf`文件的简洁，所以把 jk 模块的配置放到单独的文件中来）
&emsp;&emsp;3、`mod_jk.conf`配置内容如下：
```conf
    # Load mod_jk2 module
    LoadModule jk_module modules/mod_jk-1.2.31-httpd-2.2.3.so
    # Where to find workers.properties( 引用 workers 配置文件 )
    JkWorkersFile conf/workers.properties
    # Where to put jk logs(log 文件路径 )
    JkLogFile logs/mod_jk2.log
    # Set the jk log level [debug/error/info](log 级别 )
    JkLogLevel info
    # Select the log format(log 格式 )
    JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
    # JkOptions indicate to send SSL KEY SIZE,
    JkOptions +ForwardKeySize +ForwardURICompat -ForwardDirectories
    # JkRequestLogFormat set the request format
    JkRequestLogFormat "%w %V %T"
    # Send JSPs for context / to worker named loadBalancer(URL 转发配置，匹配的 URL 才转发到 tomcat 进行处理 )
    JkMount /*.jsp controller
    # JkMount /*.* loadBalancer
```
> 需要注意的地方是,第二行的`modules/mod_jk-1.2.31-httpd-2.2.3.so` 改为实际的mod_jk文件文件名称

&emsp;&emsp;4、`workers.properties`文件中添加内容如下：
```properties
    #server 列表
    worker.list = controller,tomcat1,tomcat2
    # tomcat1(ajp13 端口号，在tomcat下server.xml配置,默认8009)
    worker.tomcat1.port=8009
    #tomcat 的主机地址，如不为本机，请填写ip地址
    worker.tomcat1.host=localhost
    worker.tomcat1.type=ajp13
    #server 的加权比重，值越高，分得的请求越多
    worker.tomcat1.lbfactor = 1
    # tomcat2
    worker.tomcat2.port=18009
    worker.tomcat2.host=localhost
    worker.tomcat2.type=ajp13
    worker.tomcat2.lbfactor = 1
    # controller( 负载均衡控制器)
    worker.controller.type=lb
    # 指定分担请求的tomcat
    worker.controller.balanced_workers=tomcat1,tomcat2
    worker.controller.sticky_session=true
    #粘性session(默认是打开的) 当该属性值=true（或1）时，代表session是粘性的，
    #即同一session在集群中的同一个节点上处理，session不跨越节点。在集群环境中，一般将该值设置为false
    worker.controller.sticky_session=false
    #设置用于负载均衡的server的session可否共享 1为共享
    worker.controller.sticky_session_force=1
```
&emsp;&emsp;5、在`httpd.conf`文件**最后**添加如下：
 ```conf
    # JK module settings
    Include conf/mod_jk.conf
 ```

 &emsp;&emsp;6、重新启动Apache，观察是否有mod_jk的版本错误，如果有重新替换`Apache24\modules`下的`mod_jk.so`文件即可，其他配置文件无需修改；启动成功后，访问`localhost:80`即可进去Apache的主界面（与tomcat访问`localhost:8080`类似）。

### Tomcat 配置

#### 修改 server.xml 文件
 &emsp;&emsp;1、在此我准备了两个Tomcat，所以要确保个tomcat的端口不冲突，修改一下 port
 ```xml
    <Server port="9005" shutdown="SHUTDOWN">
 ```
 ```xml
    <Connector port="9080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
 ```
 ```xml
    <Connector port="9009" protocol="AJP/1.3" redirectPort="8443" />
 ```
 &emsp;&emsp;2、打开 Engine 注释，并添加 jvmRoute 属性
 ```xml
    <Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat2" >
 ```
 > 注意：jvmRoute的值要与 workers.properties中配置的相一致

&emsp;&emsp;3、打开 Cluster 注释
```xml
    <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
```
#### 修改 web.xml 文件
&emsp;&emsp;在文件末加入`<distributable/>`标签：
```xml
    …………
    <distributable/>
    </web-app>
```

## 测试

### 编写测试 MyJsp.jsp
```jsp
    <%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
    <%@ page import="java.text.SimpleDateFormat"%>
    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
    <html>
    <head>
    <title>Tomcat集群测试</title>
    </head>
    <body>
    服务器信息:
    <%
    String dtm = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss").format(new Date());
    System.out.println("["+request.getLocalAddr()+":"+ request.getLocalPort()+"]" + dtm);
    out.println("<br>["+request.getLocalAddr()+":" +request.getLocalPort()+"]" + dtm+"<br>");
    System.out.println("tomcat2");
    %>
    session分发:
    <%
    session.setAttribute("name","dennisit");
    System.out.println("[session分发] session id:"+session.getId());
    out.println("<br>[session分发] session id： " + session.getId()+"<br>");
    System.out.println("tomcat2");
    %>
    </body>
    </html>
```
### 测试效果
&emsp;&emsp;将编写好的测试程序，分别放入两个Tomcat，并启动测试保证访问成功。浏览器中输入`locahost:80/TomcatTest/MyJsp.jsp`（我自己的程序地址），多次刷新地址，页面内容如下：

![](/img/20190507-1.png)

![](/img/20190507-2.png)

&emsp;&emsp;通过访问 Apache 的服务地址，系统任意跳转到其中一个 Tomcat 进行系统展示，从而达到负载均衡的效果。
&emsp;&emsp;至此，一个基于Windows系统的简单的Tomcat集群就算是完成了。此文是自己实际操作的笔记，如有不对，多多谅解。




