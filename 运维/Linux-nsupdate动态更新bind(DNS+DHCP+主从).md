系统环境:RHEL6.2_X86_64  
实验软件:bind bind-chroot dhcp  
实验规划:192.168.0.54主DNS服务器；192.168.0.86辅助DNS。  
###### DNS服务器：(主配置文件：`/etc/nf`)
* 编辑主配置文件
```bash
[root@desktop54 ~]#  vim / etc /nf 

//    acl blacklist { 192.168.0.85; }; //可以定义变量，方便其它地方调用.
options {

        listen-on port 53 { any; }; //默认仅监听本地的端口，可以改成对应ip或”any“.
        listen-on-v6 port 53 { ::1; }; //ipv6的监听端口
        directory       "/var/named"; 
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any; };    //默认允许本地dns解析请求，可以相应的进行设定
        recursion yes;
//    blackhole { blacklist; }; //调用上面定义的访问主机.
        dnssec-enable yes;
        dnssec-validation yes;
         dns sec-lookaside auto;
        ...
        ...
zone "" IN {
        type master;    //主DNS，master
        file ".zone";    //区域文件
        allow-transfer { 192.168.0.85; }; //指定允许备份DNS数据的主机。
        allow-update { 192.168.0.85; };    //允许在辅助DNS上更新记录，用时使其生效。
};
zone "0.168.192.in-addr.arpa" IN {
        type  master ;
        file ".arpa";
        allow-transfer { 192.168.0.85; };
        allow-update { 192.168.0.85; };
};
```

* 编辑区域文件(`/var/named/`)
```bash
named.localhost(正向解析区域文件模板)
named.loopback(反向解析区域文件模板)
[root@desktop54 ~]# cd /var/named/
[root@desktop54 named]# cp -p named.localhost .zone    （-p参数，保留文件的属性）
[root@desktop54 named]# cp -p named.loopback .arpa
[root@desktop54 named]# vim .zone 
$TTL 1D
@       IN SOA  . root. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      .
        A       192.168.0.85
www     A       192.168.0.11
www     A       192.168.0.22
www     A       192.168.0.33
mail    A       192.168.0.44

[root@desktop54 named]# vim .arpa
$TTL 1D
@       IN SOA  . root. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      .
        A       192.186.0.85
66      PTR     .
77      PTR     .
88      PTR     mai.
[上面的域名没有点会出错！]
```

*     启动named服务
```bash
[root@desktop54 named]# service named start
Starting named:                                            [  OK  ]
```

* 测试DNS
在本机的`/etc/nf`文件第一行加入`nameserver 192.168.0.54`
```bash
[root@desktop54 named]# dig 
;; ANSWER SECTION:
.        86400    IN    A    192.168.0.11
.        86400    IN    A    192.168.0.22
.        86400    IN    A    192.168.0.33

[root@desktop54 named]# dig 
;; ANSWER SECTION:
.        86400    IN    A    192.168.0.33
.        86400    IN    A    192.168.0.11
.        86400    IN    A    192.168.0.22
『每一次请求 都会轮循，可以在这里做一个”负载均衡“』
[root@desktop54 named]# dig -x 192.168.0.88    (-x 反向解析)
;; ANSWER SECTION:
88.0.168.192.in-addr.arpa. 86400 IN    PTR    mai.
```
OK能正常解析！  
###### 辅助DNS备份（192.168.0.85）
* 安装软件和编辑配置文件

```bash
[root@desktop85 ~]# yum install bind bind-chroot -y
[root@desktop85 ~]# vim / etc /nf  加入以下备份区域文件，名字和主DNS服务器一样
zone "" IN {
        type slave;    //类型：辅助DNS    
        masters { 192.168.0.54; };    //指定要备份的主DNS
        file "slaves/linuxidc.zone_bak";    //备份过来的区域文件，/var/named/chroot/var/named/slaves/
};

zone "0.168.192.in-addr.arpa" IN {
        type slave;
         master s { 192.168.0.54; };
        file "slaves/linuxidc.arpa_bak";
};
```

* 启动服务、查看结果
```bash
[root@desktop85 ~]# service named start
Starting named:                                            [  OK  ]
[root@desktop85 ~]# ls /var/named/chroot/var/named/slaves/
linuxidc.arpa_bak  linuxidc.zone_bak
```
看，正向和反向区域文件备份过来了。  
* 远程修改DNS条目（保证selinux的设置允许）    在主DNS上设置selinux参数,允许写zone文件  
```bash
[root@desktop54 ~]# getsebool -a |grep named
named_write_master_zones --> off
[root@desktop54 ~]# setsebool -P named_write_ master _zones on
#在辅助DNS服务器上（因为在主DNS上设置了只有192.168.0.85这台辅助DNS可以修改条目）
[root@desktop85 ~]# nsupdate
> server 192.168.0.54
> zone 
> update add new 500 A 192.168.0.99
> send
update failed: SERVFAIL
#(还是有错呀怎么办？查看了一下是文件夹权限问题)
[root@desktop54 ～]# chown named.named /var/named/chroot/var/named/ -R
再试试～
[root@desktop85 ~]# nsupdate
> server 192.168.0.54
> zone 
> update add new 500 A 192.168.0.99        （500为ttl值）> send
> quit
[root@desktop85 ~]# dig new
;; ANSWER SECTION:
new.        500    IN    A    192.168.0.99    (就是刚刚设置的ip 99)
[root@desktop54 ~]# ls /var/named/chroot/var/named/ | grep jnl
.zone.jnl    (就是同步过来产生的文件啦)
『nsupdate用法看man文档，可以加密传输的。下面有创建密钥的步骤』
```

###### DHCP
* 主DNS（192.168.0.54）上编辑DHCP的主配置文件拷贝一份模板配置文件
```bash
[root@desktop54 ~]# cp /usr/share/doc/dhcp-nf.sample /etc/dhcp/nf
[root@desktop54 ~]# vim / etc /dhcp/nf 
ddns-update-style interim;
ignore client-updates;
subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.110 192.168.0.112;
  option domain-name-servers 192.168.0.54;
  option domain-name "";
  option routers 192.168.0.54;
  option broadcast-address 192.168.0.54;
  default-lease-time 600;
  max-lease-time 7200;
}
```
* 启动服务

```bash
[root@desktop54 ~]# service dhcpd start
Starting dhcpd:                                            [  OK  ]
```

* DHCP和DNS都配置好了，测试一下  ip自动获取,将刚刚的 192.168.0.85 的网络配置文件 ip改为自动获取 `BOOTPROTO="dhcp"` DNS改为 192.168.0.54
```bash
[root@desktop85 ~]# vim /etc/nf 
nameserver 192.168.0.54
```
网络重启(为了不影响实验结果，将网线拔掉吧)
```bash
[root@desktop85 ~]# service network restart
[root@desktop85 ~]# ifconfig eth0 |grep 'inet addr'|awk '{print $2}'|cut -d: -f2
192.168.0.111
(获取到ip 192.186.0.111)

[root@desktop85 ~]# cat /etc/nf 
; generated by /sbin/dhclient-script    （有dhcp脚本生成）
search     (域名)
nameserver 192.168.0.54    （DNS服务器）

再看看DHCP租约文件：
[root@desktop54 ~]# cat /var/lib/dhcpd/dhcpd.leases
lease 192.168.0.111 {
  starts 1 2012/03/12 19:23:38;
  ends 1 2012/03/12 19:33:38;
  tstp 1 2012/03/12 19:33:38;
  cltt 1 2012/03/12 19:23:38;
  binding state free;
  hardware ethernet 52:54:00:16:73:bb;
}

对比192.168.0.111主机的MAC：
[root@desktop85 ~]# grep 'ATTR' /etc/udev/rules.d/70-persistent-net.rules |awk -F\" '{print $8}'
52:54:00:16:73:bb    （和上面的申请记录一样吧，说明DHCP成功了哦）
```
OK 这样就整合完成，DDNS也算配置好了。

###### 配置安全的DDNS
* 创建密钥（主服务器上操作）
```bash
[root@desktop54 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST linuxidc 
Klinuxidc.+157+46472
『 dns sec-keygen：用来生成更新密钥。-a HMAC-MD5：采用HMAC-MD5加密算法。-b 128：生成的密钥长度为128位。-n HOST linuxidc：密钥主机linuxidc。』

[root@desktop54 ~]# ls Klinuxidc*
Klinuxidc.+157+46472.key  Klinuxidc.+157+46472.private （公钥和私钥）

[root@desktop54 ~]# cat Klinuxidc.+157+46472.key Klinuxidc.+157+46472.private 
linuxidc. IN KEY 512 3 157 YgSc3M4mBNCkTkA328kdxA==    （公钥）
Private-key-format: v1.3    （私钥）
Algorithm: 157 (HMAC_MD5)
Key: YgSc3M4mBNCkTkA328kdxA==
Bits: AAA=
Created: 20120312205523
Publish: 20120312205523
Activate: 20120312205523
```

* 将密钥加到DNS主配置文件   
查看模板：  
```bash
[root@desktop54 ~]# cat /etc/rndc.key 
key "rndc-key" {
    algorithm hmac-md5;
    secret "Vrjz2o7eevbbmtp/6CeT6Q==";
};
```
定义密钥  
```bash
[root@desktop54 ~]# vim /etc/nf     (添加下面的密钥)
key "linuxidc"
{
  algorithm hmac-md5;    （密钥的算法：hmac-md5）
  secret "YgSc3M4mBNCkTkA328kdxA==";     （密钥）
};
```
添加密钥  
将的`allow-update { 192.168.0.85; }` 中的IP换为`“key linuxidc”`；  
将`0.168.192.in-addr.arpa`中的`allow-update { 192.168.0.85; }`中的IP换为`"key linuxidc"`;  
将`allow-transfer { 192.168.0.85; }`中的IP换为`“key linuxidc”`；允许远程登录修改。  
之前是使用IP更新区域文件，但是IP可以伪造的，不是很安全。所以设定为只让拥有密钥的用户才可以更新。  
重启服务`[root@desktop54 ~]# service named restart`  

* 将密钥加到DHCP主配置文件
```bash
[root@desktop54 ~]# vim /etc/dhcp/nf 
key "linuxidc"
{
  algorithm hmac-md5;
  secret "YgSc3M4mBNCkTkA328kdxA==";    
}    (注意这里结尾没有分号；，和named配置不一样)

zone .    （注意这里有个点.）
{       primary 127.0.0.1;
        key linuxidc;
}    （注意这里结尾没有分号；，和named配置不一样）
zone 0.168.192.in-addr.arpa.    (注意这里有个点.)
{
        primary 127.0.0.1;
        key linuxidc;
}    （注意这里结尾没有分号；，和named配置不一样）

[root@desktop54 ~]# service dhcpd restart
Shutting down dhcpd:                                       [  OK  ]
Starting dhcpd:                                            [  OK  ]
[root@desktop54 ~]# service named restart
Stopping named:                                            [  OK  ]
Starting named:                                            [  OK  ]
```

* 测试DDNS  
```bash
#先查看192.168.0.85的主机名
[root@desktop85 ~]# hostname 
desktop8

#在85主机上新建一个名为“nf”的dhcp客户端配置文件
[root@desktop85 ~]# vim /etc/dhcp/nf
send fqdn.fqdn "dhcp8";
send fqdn.encoded on;
send fqdn.server-update on;
[root@desktop85 ~]# service network restart
#测试
[root@desktop54 ~]# dig dhcp8
;; ANSWER SECTION:
dhcp8.    300    IN    A    192.168.0.111   (111就是获取到的IP地址啦)
这样当获得的动态ip也能互相查到了。就不用手工录入解析。

[root@desktop54 ~]# cat /var/lib/dhcpd/dhcpd.leases
lease 192.168.0.111 {
  starts 1 2012/03/12 22:42:26;
  ends 1 2012/03/12 22:52:26;
  cltt 1 2012/03/12 22:42:26;
  binding state active;
  next binding state free;
  hardware ethernet 52:54:00:16:73:bb;
  set ddns-fwd-name = "dhcp8";    (就是这儿了)
  set ddns-txt = "0036a33e6d9f5120ed2dbf20dbd50d636b";
  set ddns-rev-name = "111.0.168.192.in-addr.arpa.";    （对应的IP地址）
}
[root@desktop54 ~]# nslookup dhcp8    (dig一下效果也是一样呢)
Server:        192.168.0.54    （通过54主机解析）
Address:    192.168.0.54#53    （53端口）

Name:    dhcp8
Address: 192.168.0.111

```

这样DDNS的功能就呈现出来了～～




