magent是一款开源的memcached代理服务器软件  
地址： http://code.google.com/p/memagent/  
安装magent到/usr/local/下  
```bash
cd /usr/local
mkdir magent
cd magent/
wget    http://memagent.googlecode.com/files/magent-0.5.tar.gz
tar zxvf magent-0.5.tar.gz
/sbin/ldconfig
sed -i "s/LIBS = -levent/LIBS =-levent -lm/g" Makefile
make
```

github 上搜索 memagent   然后 修改makerfile 安装  
  

```bash
magent命令参数：
-hthis message
-u uid
-g gid
-p port, default is 11211. (0 to disable tcpsupport)
-s ip:port, set memcached server ip and port
-b ip:port, set backup memcached server ip andport
-l ip, local bind ip address, default is 0.0.0.0
-n number, set max connections, default is 4096
-D don't go to background
-k use ketama key allocation algorithm
-f file, unix socket path to listen on. defaultis off
-i number, max keep alive connections for onememcached server, default is 20
-v verbose

#启动magent服务
magent -u root -n 4096 -l 127.0.0.1 -p12000           -s127.0.0.1:8086           -s 127.0.0.2:8086            -b 127.0.0.1:11213
```

Magent使用举例  
启动两个memcached进程，端口分别为11211和11212：  
```bash
memcached -m 1 -u root -d -l 127.0.0.1 -p 11211
memcached -m 1 -u root -d -l 127.0.0.1 -p 11212
```

启动两个magent进程，端口分别为10000和11000：  
```bash
magent -u root -n 51200 -l 127.0.0.1 -p 10000 -s127.0.0.1:11211 -b 127.0.0.1:11212
magent -u root -n 51200 -l 127.0.0.1 -p 11000 -s127.0.0.1:11212 -b 127.0.0.1:11211

#-s 为要写入的memcached， -b 为备份用的memcached。
```

说明：测试环境用magent和memached的不同端口来实现，在生产环境中可以将magent和memached作为一组放到两台服务器上。  
也就是说通过magent能够写入两个memcached。