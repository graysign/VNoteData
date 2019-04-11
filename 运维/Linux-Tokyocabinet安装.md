Tokyo Cabinet 是日本人Mikio Hirabayashi 开发的一款DBM 数据库，该数据库读写很快。哈希模式写入100 万条数据仅仅需0.643 秒。读取100 万条数据仅仅需0.773 秒，是Berkeley DB 等DBM 的几倍。
Tokyo Tyrant 是由同一作者开发的Tokyo Cabinet 数据库网络接口。它拥有Memcached兼容协议，也能够通过HTTP 协议进行数据交换。
Tokyo Tyrant 加上Tokyo Cabinet，构成了一款支持高并发的分布式持久存储系统，对不论什么原有Memcached client来讲，能够将Tokyo Tyrant 看成是一个Memcached。可是，它的数据是能够持久存储的。

安装:  
```bash
git clone https://github.com/maiha/tokyocabinet.git
./configure --enable-off64 --prefix=/u01/tokyocabinet/
make
make install



#在编译 tokyocabinet 时会报 configure: error: bzlib.h is required 的错误。
#解决方法是：   yum install bzip2-devel
```
```bash
git clone https://github.com/maiha/tokyotyrant.git
 ./configure --prefix=/u01/tokyotyrant --with-tc=/u01/tokyocabinet/
make
make install
```


启动:  
```bash
#启动tt最简单的方式，直接输入命令
 
    ttserver

#    能够看到默认使用1978port，监听全部地址。
```