title: Redis报错
date: '2019-09-03 23:12:50'
tags: [redis]
categories: [工作笔记]
---
项目中在使用 Redis 时遇到如下报错

```java
······
Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisException: Could not get a resource from the pool
······
DENIED Redis is running in protected mode because protected mode is enabled
```
<!-- more -->

## 解决方案
### 百度到的- 修改redis服务器的配置
+ 编辑配置文件

```vim
vi redis.conf 
```
+ 注释以下绑定的主机地址

```conf
# bind 127.0.0.1
```

+ 修改redis的守护线程为`no`，不启用

```vim
127.0.0.1:6379> config set daemonize "no"
OK
```

+ 修改redis的保护模式为`no`，不启用

```vim
127.0.0.1:6379> config set protected-mode "no"
OK
```

> 如果redis实例配置文件中禁用了bind参数，并将protected-mode设置为no后，外网访问redis依然报上述错误，因为 sentinel 实例的配置文件中需要增加参数 protected-mode  no

### 问题解决
&emsp;&emsp;根据百度经验查看了上述的配置之后，发现该设置的地方都已经设置了，但是结果还是访问报错；

+ Redis启动

1、直接启动，进入redis根目录，执行命令:

```sh
./redis-server &
```
> 加上‘&’号使redis以后台程序方式运行

2、通过指定配置文件启动

```sh
./redis-server ../redis.conf
```

&emsp;&emsp;正常的启动应该是没有问题，但是我的坑就在这里，指定配置文件启动也是不行；因为`相对路径` 的原因；在换了执行命令`./redis-server /user/local/redis/redis.conf` 后就能正常访问了；