**目录**

*   [RTMP协议简介](#_label0)
*   [RTMP服务器](#_label1)
*   [RTMP推流器](#_label2)

* * *

由于项目需要 RTMP 推送 H264 数据，在网上查找了下相关的方案，总结一下。


RTMP协议简介
--------

在总结之前，我们先简单介绍一下 RTMP 协议。  
RTMP(Real Time Messaging Protocol) 实时消息传送协议是 Adobe Systems 公司为 Flash 播放器和服务器之间音频、视频和数据传输开发的私有协议。

它有三种变种：

1）工作在 TCP 之上的明文协议，使用端口 1935；

2）RTMPT 封装在 HTTP 请求之中，可穿越防火墙；

3）RTMPS 类似 RTMPT，但使用的是 HTTPS 连接；

RTMP 协议就像一个用来装数据包的容器，这些数据可以是 AMF 格式的数据,也可以是 FLV 中的视/音频数据。一个单一的连接可以通过不同的通道传输多路网络流。这些通道中的包都是按照固定大小的包传输的。

更多协议的细节可以参见《rtmp specification 1.0》。

  

RTMP服务器
-------

RTMP 服务器，现成的开源方案有以下几种推荐：

1\. nignx

Nginx("engine x") 是一款是由俄罗斯的程序设计师 Igor Sysoev 所开发高性能的 Web 和反向代理 服务器，也是一个 IMAP/POP3/SMTP 代理服务器。在高连接并发的情况下，Nginx 是 Apache 服务器不错的替代品。

我们这里使用的是 Nginx 的 rtmp 插件实现实时流推送，具体实现可以参考我的另一篇博客：[Windows 搭建 nginx RTMP 服务器](https://www.cnblogs.com/linuxAndMcu/p/12517787.html)

  

2\. srs

[SRS](http://ossrs.net/)(Simple RTMP Server) 是国人写的一款非常优秀的开源流媒体服务器软件，使用 C++ 开发，可用于直播/录播/视频客服等多种场景，其定位是运营级的互联网直播服务器集群。SRS 提供了丰富的接入方案将 RTMP 流接入 SRS，包括推送 RTMP 到 SRS、推送 RTSP/UDP/FLV 到 SRS、拉取流到 SRS。SRS 还支持将接入的 RTMP 流进行各种变换，譬如将 RTMP 流转码、流截图、转发给其他服务器、转封装成 HTTP-FLV 流、转封装成 HLS、转封装成 HDS、录制成 FLV。GitHub 源码链接为：[https://github.com/ossrs/srs](https://github.com/ossrs/srs)

  

3\. crtmpserver c++

crtmpserver 是一个由 C++ 语言编写的开源的 RTMP 流媒体服务器，与其对应的商业产品自然是 Adobe 公司的 FMS。与 FMS 相比，从功能上来说crtmpserver 只能称为 FMS 的简化版本，其功能并没有 FMS 那么完善甚至是远远没有达到。其与 flash player 的兼容性自然也比不上官方的 FMS 了。但是 crtmpserver 提供了最常见的 RTMP 实现。作为开源的高性能 RTMP 流媒体服务器，不仅可以用在 x86 平台的 linux 服务器，windows 服务器，还可以被用在 arm 等嵌入式平台上。crtmpserver 的代码结构良好，类的继承体系清楚，代码效率高。是学习 RTMP 协议和服务器端编程的好例子。GitHub 源码链接为：[https://github.com/shiretu/crtmpserver](https://github.com/shiretu/crtmpserver)

crtmpserver 源码依赖 openssl，所以不管是在 Linux 还是 Windows 平台下，都需要先编译 openssl 库，具体编译请参考：[crtmpserver系列(二)：搭建简易流媒体直播系统](https://www.cnblogs.com/wangqiguo/p/6014519.html)

  

4\. livego

[Go](https://golang.org/)（又称Golang，[wiki](https://en.wikipedia.org/wiki/Go_(programming_language)) [中文](https://zh.wikipedia.org/wiki/Go)）是Google开发的一种静态强类型、编译型、并发型，并具有垃圾回收功能的开源编程语言（[github](https://github.com/golang/go)），支持windows、linux、macOS等操作系统。

livego 是基于 go 语言的 rtmp 直播服务器。go 语言为服务器性能而生，开发效率远远高过 C/C++。GitHub 源码链接为：[https://github.com/gwuhaolin/livego](https://github.com/gwuhaolin/livego)

**为什么基于golang?**

go 在语言基本支持多核 CPU 均衡使用，支持海量轻量级线程，提高其并发量。当前开源的缺陷：

*   srs 只能运行在一个单核下，如果需要多核运行，只能启动多个 srs 监听不同的端口来提高并发量；
*   ngx-rtmp 启动多进程后，报文在多个进程内转发，需要二次开发，否则静态推送到多个子进程，效能消耗大；

go 在语言级别解决了上面多进程并发的问题。具体请参考：[默默前行的livego--基于go语言的rtmp直播服务器](https://www.cnblogs.com/runner42/p/7248974.html)

  

5\. node-rtsp-rtmp-server

Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境。 Node.js 使用了一个事件驱动、非阻塞式 I/O 的模型。

node-rtsp-rtmp-server 是使用 Node.js 实现的 rtmp 服务器。GitHub 源码链接为：[https://github.com/iizukanao/node-rtsp-rtmp-server](https://github.com/iizukanao/node-rtsp-rtmp-server)

  

测试

测试的话下载个推流工具，建议使用[大牛直播](https://www.daniulive.com/index.php/sdk-demo%e4%b8%8b%e8%bd%bd/)提供的推流工具，也可以使用 FFmpeg 推流。

  

[回到顶部](#_labelTop)

RTMP推流器
-------

1\. librtmp

RTMPDump 软件包含一个基本的客户端：[rtmpdump](http://rtmpdump.mplayerhq.hu/)，一些示例服务器和一个用来提供对RTMP协议进行支持的库（libRTMP）。librtmp 使用的人非常多，但也比较老了，具体请参考：

[H264视频通过RTMP直播](https://blog.csdn.net/firehood_/article/details/8783589)

[windows - librtmp推送H264的Demo下载](https://files.cnblogs.com/files/dong1/librtmp_pusher_win.zip)

[最简单的基于librtmp的示例：发布H.264（H.264通过RTMP发布）](http://blog.csdn.net/leixiaohua1020/article/details/42105049)

  

2\. FFmpeg

FFmpeg 也能实现 rtmp 推流，因为内部集成了 librtmp，官方给出了 muxing.c 源代码，就是实现如何推流的例子。具体可以参考：

[ffmpeg 代码实现rtmp推流到服务器](https://blog.csdn.net/zhangpengzp/article/details/89713422)

[《最简单的基于FFmpeg的推流器（以推送RTMP为例）》](http://blog.csdn.net/leixiaohua1020/article/details/39803457)

[基于FFmpeg进行RTMP推流（一）](https://www.jianshu.com/p/69eede147229)

[Windows远程桌面实现之五（FFMPEG实现桌面屏幕RTSP,RTMP推流及本地保存）](https://blog.csdn.net/fanxiushu/article/details/80996391)

  

3\. srs-librtmp

srs-librtmp 是 srs 提供的一个 rtmp 库，可以推送 H264 数据。本来打算使用 srs 来做 rtmp 推流器，但是 srs 只提供了 Linux 平台的 srs\_librtmp.lib 静态库和 Demo，费了些功夫，修修改改成功编译了 Windows 下的 srs\_librtmp.lib 静态库，但是添加进工程后，工程可以打开库，却识别不了所调用的 API，由于时间问题暂时放弃。

  

**参考：**

[rtmp服务器以及rtmp推流/拉流/转发](https://www.cnblogs.com/dong1/p/10200869.html)