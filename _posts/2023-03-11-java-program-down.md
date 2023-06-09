---
title: 一次 Java 程序假死的排查过程
key: a_20230311_01
tags: Java
modify_date: 2023-03-12
---

出差回来同事离职，他手里的项目交接给我。工作两年时间，第一次遇到 Java 进程假死的情况，毫无头绪的时候 ChatGPT 帮了大忙，记录一下解决问题的过程。

<!--more-->

## 一、项目背景

项目是使用 SpringBoot + Netty 编写的一个 TCP 客户端程序，服务端是另一个公司提供，双方基于联通 C 接口协议进行数据交互，在程序运行了大概半个月后出现了假死的情况。

## 二、初步排查

接到程序假死的通知后，立马登上服务器，把 GC 信息、Jmap 信息以及程序堆栈信息导出，然后重启了进程，重启后程序恢复正常。

```sh
# 堆栈信息
jstack 进程ID > 文件名.txt

# Jmap信息
jmap -dump:format=b,file=文件名.bin 进程ID

# GC信息（单位：毫秒）
jstat -gcutil 进程ID 时间间隔 查询次数 > 文件名.txt
```

重启恢复正常初步判断应该是由于一些资源没有释放，长时间运行慢慢积累最后把内存撑爆了，所以重启后就恢复了。但是到这里肯定还没有结束，如果不解决这个问题，再长时间运行肯定会再次出现问题。

先看了下堆栈信息，没有看出什么问题。又把目光转向 GC 信息，好家伙，新生代老年代内存都满了，Full GC 竟然高达两万次以上，连续抓取的几次 GC 信息里都没有进行垃圾回收。

![GC](/images/20230311/yFlJJpq.jpeg)

接下来就是找程序中的问题。首先看了报错日志，有解码时下标越界的问题，还有 redis 空指针的问题。

看了解码那部分代码，感觉是同事没有理解协议中报文格式的定义以及 Bytebuf 的概念，这块倒是问题不大。花了半个小时重写了解码器，因为之前的电信 C 接口也是相同的协议，只是报文格式不同。

至于 redis 这块，有点诡异，因为在 if 条件里判断了如果 key 存在，才会取这个 key，但是取的时候竟然空指针了。没想明白问了架构师，因为程序是多线程，并发量比较大，他建议我把 redis 客户端由 jedis 换成 lettuce 并启用连接池。

换完客户端之后打包进行部署，为了能尽早暴露问题，我把 Jvm 的 Xmx 参数设置为 2G。经过修改程序不报错了，又持续观察了一下午的 GC 情况，虽然老年代持续增长但是涨幅很慢。当时以为是 redis 连接没释放导致的问题，所以当天就这样结束了，下面是当天更新后的程序 GC 情况。

![更新后的GC情况](/images/20230311/image-20230314224007706.png)

## 三、再次宕机

本以为事情已经结束了，没想到第二天一早就收到项目经理的消息说程序又没数据了，我上去一看，果然老年代又满了，但是没有触发 GC。

这次不敢大意了，立马又导出了 GC 信息和 Jmap 信息，然后在 crontab 里写了定时任务，一个小时重启一次程序😂，留给我足够的时间去排查问题。

还记得第一次宕机时导出的 Jmap 信息吗？因为它将近 31G 的大小，所以没有办法下载下来进行分析，后来询问 ChatGPT 得知使用 jhat 命令可以在服务器端生成一个 web 页面。执行 `jhat xxx.bin` 命令，等了大概五分钟后 31G 的 Jmap 文件解析完成并在 7000 端口提供了一个在线分析程序。通过实例数从大到小排列，终于看到了大量占用空间的实例数，netty 相关的包实例数加起来过百万。

![instance](/images/20230311/WFc6XPf.png)

突然想到对方服务端最近一直出问题登录失败，导致我们客户端一直断线重连，会不会是断线重连这块出了问题？

![断线重连](/images/20230311/FlsMkCd.png)

由于对 Netty 不是很熟悉，所以我把断线重连的代码贴在了 v2ex 上，又问了下 ChatGPT，最后两个渠道给我的反馈都指向了 EventLoopGroup 没有被正确关闭。

![v2ex](/images/20230311/image-20230314230734953.png)

![ChatGPT](/images/20230311/image-20230314230823557.png)

我仔细检查了断线重连的代码逻辑，发现确实是每次重连都新建了 EventLoopGroup 这个线程池，没有进行复用也没有进行关闭。为了验证这个问题我在本地 IDE 启动了项目，Xmx参数设置为 25M，所有业务逻辑全部注释掉，只做断线重连操作，一秒一次，在程序运行了一分钟后，控制台打印了 OOM。

找到问题就好办，修改 client 类为单例模式，将 connect 方法中的 EventLoopGroup 和 BootStrap 提取进行复用。

![修改后的断线重连](/images/20230311/image-20230314232408714.png)

再次启动 IDE，这次 25M 内存就可以无限次断线重连了。再次打包上传，去掉定时重启，启动新程序。

## 四、持续观察

后面连续三天程序没有再出现假死的情况，每天我都观察了 GC 的情况，Full GC 次数显著降低，而且在 Full GC 时可以正常回收老年代的内存，到这里问题应该是解决了。

![3月6日GC情况](/images/20230311/image-20230314232842565.png)

![3月7日GC情况](/images/20230311/image-20230314232910549.png)

![3月8日GC情况](/images/20230311/image-20230314232924795.png)
