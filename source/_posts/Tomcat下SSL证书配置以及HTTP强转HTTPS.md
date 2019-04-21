title: Tomcat下SSL证书配置以及HTTP强转HTTPS
date: '2019-03-12 18:05:32'
tags: [SSL, Tomcat, HTTP转HTTPS]
categories: [课余笔记]
---
&emsp;&emsp;登，等灯………，首先恭喜一下自己域名成功备案😉，然后接下来自然是迫不及待绑定自己的博客啦。那么问题来了，输入域名访问后，浏览器提示“不安全”，这不能忍啊🙃。很显然这就是HTTP和HTTPS的区别😏!

<!-- more -->

## HTTP与HTTPS
&emsp;&emsp;HTTP：是一种详细规定了浏览器和万维网服务器之间互相通信的规则，通过因特网传送万维网文档的数据传送协议。
&emsp;&emsp;HTTPS：
* 以安全为目标的HTTP通道，（简而言之HTTP的安全版）。
* HTTPS的安全基础是SSL,(SSL用于加密)。
* 是一个URI schema（抽象标识符体系）。

> 两者区别：HTTPS协议需要CA申请证书，一般免费证书很少，需要交费。HTTP是超文本传输协议，信息是明文传输，HTTPS则是具有安全性的ssl加密传输协议。且两者使用的是完全不同的连接方式，其端口也不一样，**HTTP是80，HTTPS是443**。

## SSL证书

&emsp;&emsp;所以想用HTTPS前提你得有个证书，我是在阿里云购买的免费证书😆。获取方式：[云盾证书服务](https://common-buy.aliyun.com/?spm=5176.2020520163.cas.1.zTLyhO&commodityCode=cas#/buy)。
![](/img/20190312-1.png)
&emsp;&emsp;直接购买，购买成功后可在[SSL管理控制台](https://yundunnext.console.aliyun.com/?spm=5176.8050866.1280361.aliyun-search-item.4b724823XWgCDn&p=cas&accounttraceid=d9aada0f-1264-4b69-86b9-6734747453ba#/overview/cn-hangzhou)查看证书状态。
![](/img/20190312-2.png)
&emsp;&emsp;点击申请，填写相关信息，然后等待审核（审核速度很快哦）。通过之后可在“已签发”页签下，下载相关证书（根据自己需要下载）；![](/img/20190312-3.png)
## SSL证书Tomcat设置
&emsp;&emsp;这里我用的Tomcat9，所以下载的是Tomcat下的证书类型。证书下载之后应该包含两个文件：一个是 . pfx文件；一个是password文件。Tomcat支持JKS格式的证书，从Tomcat7之后也支持PFX格式证书。
&emsp;&emsp;1、在Tomcat的安装目录下创建一个cert文件夹，将 . pfx文件传入文件夹；
&emsp;&emsp;2、修改conf文件夹下server.xml文件，找到`connection port='8443'`,(原先应该被注释掉了，取消注释)，修改如下：
```xml
   <Connector port="443" 
                     protocol="org.apache.coyote.http11.Http11NioProtocol" 
                     maxThreads="150"
                     SSLEnabled="true" 
                     scheme="https" secure="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="cert/****.pfx" //证书地址 
              certificateKeystoreType="PKCS12" 
              certificateKeystorePassword="****" />//证书密钥
        </SSLHostConfig>
    </Connector>
```
&emsp;&emsp; ***部分自行填写。（可能是Tomcat 9 或者是适用JKS证书格式的原因，有些`<Connector>`标签下没有`<SSLHostConfig>`标签,大家或者可以尝试以下方法。）
```xml
  <Connector port="443" 
              protocol="org.apache.coyote.http11.Http11Protocol" 
              SSLEnabled="true"
              maxThreads="150" scheme="https" secure="true"
              keystoreFile="conf/***.jks" //证书地址
              keystorePass="***" //证书密钥
              clientAuth="false" sslProtocol="TLS" />
```
&emsp;&emsp;3、除此之外，还需要修改两处端口为443；
```xml
    <Connector port="80" protocol="HTTP/1.1"  connectionTimeout="20000"
              redirectPort="443" />
```
```xml
    <Connector port="8009" protocol="AJP/1.3" redirectPort="443" />
```
&emsp;&emsp;4、在启动服务之前，还有一点需要注意，开放防火墙的443端口；
&emsp;&emsp;5、至此访问 htttps:// 就可以成功访问了；
## HTTP强转HTTPS
&emsp;&emsp;做完以上几点，还不能实现http强转到https。
&emsp;&emsp;在阿里云里有种方法，[HTTP重定向至HTTPS](https://help.aliyun.com/document_detail/69312.html?spm=5176.10695662.1996646101.searchclickresult.3e0874f74ytFSe)，有条件的可以试一下😂。还有一种针对Tomcat的方法，可以在conf文件夹下的web.xml文件中进行如下配置：
&emsp;&emsp;在`<welcome-file-list>`后面加上下面这段：
```xml
<security-constraint>  
    <web-resource-collection >  
        <web-resource-name >OPENSSL</web-resource-name>  
        <url-pattern>/*</url-pattern>  
    </web-resource-collection>  
    <user-data-constraint>  
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>  
    </user-data-constraint>  
</security-constraint>
```
&emsp;&emsp;不理解标签含义吗？看这里[web.xml](https://liuna718-163-com.iteye.com/blog/2217057)
&emsp;&emsp;这样在访问 http:// 时，就会自动跳转到 https:// 。
&emsp;&emsp;可能还有其他没有考虑到的因素，多多体谅，大家可以一起探讨。我在操作的时候参考了:https://www.jianshu.com/p/086817e78acd  ,大家也可以参考一下。