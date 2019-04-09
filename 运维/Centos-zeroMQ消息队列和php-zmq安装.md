ZeroMQ是个类似于Socket的一系列接口，他跟Socket的区别是：普通的socket是端到端的（1:1的关系），而ZMQ却是可以N：M 的关系，人们对BSD套接字的了解  
较多的是点对点的连接，点对点连接需要显式地建立连接、销毁连接、选择协议（TCP/UDP）和处理错误等，而ZMQ屏蔽了这些细节，让你的网络编程更为简单。  
ZMQ用于node与node间的通信，node可以是主机或者是进程。  
ZeroMQ是网络栈中新的一层，它是个可伸缩层，分散在分布式系统间。因此，它可支持任意大的应用程序。ØMQ不是简单的点对点交互，相反，它定义了分布式系统  
的全局拓扑。ØMQ应用程序没有锁，可并行运行。此外，它可在多个线程、内核和主机盒之间弹性伸缩。与其他的消息队列相比，ZeroMQ有以下一些特点  

* 点对点无中间节点  
传统的消息队列都需要一个消息服务器来存储转发消息。而ZeroMQ则放弃了这个模式，把侧重点放在了点对点的消息传输上，并且（试图）做到极致。以为消息  
服务器最终还是转化为服务器对其他节点的点对点消息传输上。ZeroMQ能缓存消息，但是是在发送端缓存。ZeroMQ里有水位设置的相关接口来控制缓存量。当然，  
ZeroMQ也支持传统的消息队列（通过zmq_device来实现）。
* 强调消息收发模式  
在点对点的消息传输上ZeroMQ将通信的模式做了归纳，比如常见的订阅模式（一个消息发多个客户），分发模式（N个消息平均分给X个客户）等等。下面是目前支持  
的消息模式配对，任何一方都可以做为服务端。
    * PUB and SUB
    * REQ and REP
    * REQ and XREP
    * XREQ and REP
    * XREQ and XREP
    * XREQ and XREQ
    * XREP and XREP
    * PUSH and PULL
    * PAIR and PAIR
* 以统一接口支持多种底层通信方式（线程间通信，进程间通信，跨主机通信）  
  如果你想把本机多进程的软件放到跨主机的环境里去执行，通常要将IPC接口用套接字重写一遍。非常麻烦。而有了ZeroMQ就方便多了，只要把通信协议从  
  `ipc:///xxx`改为`tcp://*.*.*.*:****`就可以了，其他代码通通不需要改，如果这个是从配置文件里读的话，那么程序就完全不要动了，直接复制到其他机器上  
  就可以了。以为ZeroMQ为我们做了很多。
* 异步，强调性能  
ZeroMQ设计之初就是为了高性能的消息发送而服务的，所以其设计追求简洁高效。它发送消息是异步模式，通过单独出一个IO线程来实现，所以消息发送调用之后不要  
立刻释放相关资源哦，会出错的（以为还没发送完），要把资源释放函数交给ZeroMQ让ZeroMQ发完消息自己释放。zeromq的官方网站: http://www.zeromq.org/  

```bash
系统环境约定:
OS：CentOS 6.3 X64
web目录：/htdoc/web
软件临时存放目录：/opt/
zeromq安装目录：/usr/local/webserver/zeromq
```
###### 安装zeromq
```bash
#Tip:To build on UNIX-like systems,make sure that libtool, autoconf, automake are installed.
[root@iredmail opt]# wget http://download.zeromq.org/zeromq-3.2.3.tar.gz
[root@iredmail opt]# tar -zxvf zeromq-3.2.3.tar.gz
[root@iredmail opt]# cd zeromq-3.2.3
[root@iredmail zeromq-3.2.3]# ./configure --prefix=/usr/local/webserver/zeromq --with-pgm=libpgm-5.1.118~dfsg
(libpgm-5.1.118~dfsg位于zeromq-3.2.3/foreign/openpgm/目录下)
[root@iredmail zeromq-3.2.3]# make && make install
[root@iredmail zeromq-3.2.3]# ldconfig
#官方文档：http://www.zeromq.org/area:download
#Tip：If you get an error：configure: "error: cannot link with -luuid, install uuid-dev".Please install e2fsprogs-devel
```
###### 安装PHP扩展
```bash
[root@iredmail zeromq-3.2.3]# cd..
[root@iredmail opt]# git clone git://github.com/mkoppanen/php-zmq.git
Tip: If you get an error of git command，please install git or use other download tools
[root@iredmail opt]# cd php-zmq
[root@iredmail php-zmq]# /usr/local/webserver/php5318/bin/phpize
#Tip: 
#①/usr/local/webserver/php5318/is my php installation path;
#②If you are using php installed from your distribution's package manager the 'phpize' command is usually in php-dev or php-devel package
[root@iredmail php-zmq]# ./configure --with-php-config=/usr/local/webserver/php5318/bin/php-config --with-zmq=/usr/local/webserver/zeromq
#Tip: 
#①/usr/local/webserver/zeromq is my zeromq installation path;
#②If you get an error："checking libzmq installation... configure: error: Unable to find libzmq installation",please use param"--with-zmq" specify the zeromq installation path.
[root@iredmail php-zmq]# make && make install
Installing shared extensions:     /usr/local/webserver/php5318/lib/php/extensions/no-debug-non-zts-20090626/
#表示生成了动态链接库文件zmq.so.这个时候可以查看一下目录里有没有zmq.so 这个文件．
#Add the following line to your php.ini:
#extension=zmq.so
#OR:
#If you are using PHP 5.4.x and/or using PHP-FPM, you will need to add a zmq.ini file in /etc/php5/conf.d:
#Add the following:
#extension=zmq.so
#Restart php-fpm
#访问phpinfo页面就可以看到zeroMQ的消息了：
#官方文档：http://www.zeromq.org/bindings:php
#If you need an JAVA language environment，you can get client from：https://github.com/zeromq/jzmq
#If you need an Python language environment，you can get client from：https://github.com/zeromq/pyzmq
```
###### 实例测试  
ZMQ提供了三个基本的通信模型，分别是“Request-Reply”、“Publisher-Subscriber”、“Parallel Pipeline”，具体介绍可参阅官方提供的  
[《ØMQ(ZeroMQ) PHP Guide》](http://zguide.zeromq.org/php:all)，这里我们使用Request-Reply模型测试  
```bash
[root@iredmail php-zmq]# cp -a examples /htdoc/web/examples
[root@iredmail php-zmq]# cd /htdoc/web/examples
[root@iredmail examples]# php poll-server.php   //运行服务端
Added object with id o:000000000b8afe2b000000002aec4116
Received message: hello there!
#访问http://localhost/examples/client.php,会看到 string(7) "Got it!" 字样，表示写入队列成功．
[root@iredmail examples]# php client.php
string(7) "Got it!"
```
从zeromq的examples实例代码中，我们可以了解到使用ZMQ写基本的程序的方法，需要注意的是：  

* 服务端和客户端无论谁先启动，效果是相同的，这点不同于Socket。
* 在服务端收到信息以前，程序是阻塞的，会一直等待客户端连接上来。
* 服务端收到信息以后，会send一个“World”给客户端。值得注意的是一定是client连接上来以后，send消息给Server，然后Server再rev然后响应client，这种  
    一问一答式的。如果Server先send，client先rev是会报错的。
* ZMQ通信通信单元是消息，他除了知道Bytes的大小，他并不关心的消息格式。因此，你可以使用任何你觉得好用的数据格式。Xml、Protocol Buffers、  
  Thrift、json等等。
* 虽然可以使用ZMQ实现HTTP协议，但是，这绝不是他所擅长的。  

检查zeroMQ状态  
现在可以查看是否监听了 5555 端口  
```bash
[root@iredmail php-zmq]# lsof -i:5555
COMMAND  PID USER   FD   type DEVICE SIZE/OFF NODE NAME
php-fpm 1047  www   11u  IPv4  93051      0t0  TCP iredmail:58208->iredmail:personal-agent (ESTABLISHED)
php-fpm 1048  www   11u  IPv4  93047      0t0  TCP iredmail:58206->iredmail:personal-agent (ESTABLISHED)
php-fpm 1051  www   11u  IPv4  93049      0t0  TCP iredmail:58207->iredmail:personal-agent (ESTABLISHED)
php     1893 root    9u  IPv4  93046      0t0  TCP iredmail:personal-agent (LISTEN)
php     1893 root   10u  IPv4  93048      0t0  TCP iredmail:personal-agent->iredmail:58206 (ESTABLISHED)
php     1893 root   11u  IPv4  93050      0t0  TCP iredmail:personal-agent->iredmail:58207 (ESTABLISHED)
php     1893 root   12u  IPv4  93052      0t0  TCP iredmail:personal-agent->iredmail:58208 (ESTABLISHED)

[root@iredmail php-zmq]# netstat -anp | grep 5555
tcp        0      0 127.0.0.1:5555              0.0.0.0:*                   LISTEN      1893/php            
tcp        0      0 127.0.0.1:58207             127.0.0.1:5555              ESTABLISHED 1051/php-fpm        
tcp        0      0 127.0.0.1:58206             127.0.0.1:5555              ESTABLISHED 1048/php-fpm        
tcp        0      0 127.0.0.1:58208             127.0.0.1:5555              ESTABLISHED 1047/php-fpm        
tcp        0      0 127.0.0.1:5555              127.0.0.1:58206             ESTABLISHED 1893/php            
tcp        0      0 127.0.0.1:5555              127.0.0.1:58208             ESTABLISHED 1893/php            
tcp        0      0 127.0.0.1:5555              127.0.0.1:58207             ESTABLISHED 1893/php
```

更多ZeroMQ的帮助文档：  
ZMQ API参考手册：http://api.zeromq.org/  
PHP使用手册可参考：http://zguide.zeromq.org/php:all  
ZeroMQ的学习和研究：http://www.searchtb.com/2012/08/zeromq-primer.html  
ZMQ PHP编程参考手册：http://php.zero.mq  
这里有大量程序示例可供参考：https://github.com/imatix/zguide  
