###### 介绍
这篇文档旨在介绍如何安装配置基于2台服务器的MySQL集群。并且实现任意一台服务器出现问题或宕机时MySQL依然能够继续运行。  
注意！  
虽然这是基于2台服务器的MySQL集群，但也必须有额外的第三台服务器作为管理节点，但这台服务器可以在集群启动完成后关闭。同时需要注意的是并不推荐在集群启动完成后关闭作为管理节点的服务器。尽管理论上可以建立基于只有2台服务器的MySQL集群，但是这样的架构，一旦一台服务器宕机之后集群就无法继续正常工作了，这样也就失去了集群的意义了。出于这个原因，就需要有第三台服务器作为管理节点运行。  
另外，可能很多朋友都没有3台服务器的实际环境，可以考虑在VMWare或其他虚拟机中进行实验。  
下面假设这3台服务的情况：  

Server1: mysql1.vmtest.net 192.168.0.1  
Server2: mysql2.vmtest.net 192.168.0.2  
Server3: mysql3.vmtest.net 192.168.0.3  

Servers1和Server2作为实际配置MySQL集群的服务器。对于作为管理节点的Server3则要求较低，只需对Server3的系统进行很小的调整并且无需安装MySQL，Server3可以使用一台配置较低的计算机并且可以在Server3同时运行其他服务。  

###### 在Server1和Server2上安装MySQL  
从上下载`mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz`  
注意：`必须是max版本的MySQL，Standard版本不支持集群部署！`  
以下步骤需要在Server1和Server2上各做一次  

```console
mv mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz /usr/local/
cd /usr/local/
groupadd mysql
useradd -g mysql mysql
tar -zxvf mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz
rm -f mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz
mv mysql-max-4.1.9-pc-linux-gnu-i686 mysql
cd mysql
scripts/mysql_install_db --user=mysql
chown -R root  .
chown -R mysql data
chgrp -R mysql .
cp support-files/mysql.server /etc/rc.d/init.d/mysqld
chmod +x /etc/rc.d/init.d/mysqld
chkconfig --add mysqld
```
`此时不要启动MySQL！`  

###### 安装并配置管理节点服务器(Server3)  
作为管理节点服务器，Server3需要ndb_mgm和ndb_mgmd两个文件：  
从上`max-4.1.9-pc-linux-gnu-i686.tar.gz`  

```console
mkdir /usr/src/mysql-mgm
cd /usr/src/mysql-mgm
tar -zxvf mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz
rm mysql-max-4.1.9-pc-linux-gnu-i686.tar.gz
cd mysql-max-4.1.9-pc-linux-gnu-i686
mv bin/ndb_mgm .
mv bin/ndb_mgmd .
chmod +x ndb_mg*
mv ndb_mg* /usr/bin/
cd
rm -rf /usr/src/mysql-mgm
```

现在开始为这台管理节点服务器建立配置文件：  
```console
mkdir /var/lib/mysql-cluster
cd /var/lib/mysql-cluster
vi config.ini
```

在`config.ini`中添加如下内容：  
```ini
[NDBD DEFAULT]
NoOfReplicas=2
[MYSQLD DEFAULT]
[NDB_MGMD DEFAULT]
[TCP DEFAULT]
# Managment Server
[NDB_MGMD]
HostName=192.168.0.3 #管理节点服务器Server3的IP地址
# Storage Engines
[NDBD]
HostName=192.168.0.1 #MySQL集群Server1的IP地址
DataDir= /var/lib/mysql-cluster
[NDBD]
HostName=192.168.0.2 #MySQL集群Server2的IP地址
DataDir=/var/lib/mysql-cluster
# 以下2个[MYSQLD]可以填写Server1和Server2的主机名。
# 但为了能够更快的更换集群中的服务器，推荐留空，否则更换服务器后必须对这个配置进行更改。
[MYSQLD]
[MYSQLD]
```
保存退出后，启动管理节点服务器Server3：  
```console
ndb_mgmd
```
启动管理节点后应该注意，这只是管理节点服务，并不是管理终端。因而你看不到任何关于启动后的输出信息。  

###### 配置集群服务器并启动MySQL  
在Server1和Server2中都需要进行如下改动：  
```console
vi /etc/my.cnf
```

```ini
[mysqld]
ndbcluster
ndb-connectstring=192.168.0.3 #Server3的IP地址
[mysql_cluster]
ndb-connectstring=192.168.0.3 #Server3的IP地址
```
保存退出后，建立数据目录并启动MySQL：
```console
mkdir /var/lib/mysql-cluster
cd /var/lib/mysql-cluster
/usr/local/mysql/bin/ndbd --initial
/etc/rc.d/init.d/mysqld start
```
可以把`/usr/local/mysql/bin/ndbd`加到`/etc/rc.local`中实现开机启动。  

注意：`只有在第一次启动ndbd时或者对Server3的config.ini进行改动后才需要使用--initial参数！`  


###### 检查工作状态  
回到管理节点服务器Server3上，并启动管理终端：  
```console
/usr/bin/ndb_mgm
```
键入show命令查看当前工作状态：（下面是一个状态输出示例）  
```console
[root@mysql3 root]# /usr/bin/ndb_mgm
-- NDB Cluster -- Management Client --
ndb_mgm> show
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @192.168.0.1  (Version: 4.1.9, Nodegroup: 0, Master)
id=3    @192.168.0.2  (Version: 4.1.9, Nodegroup: 0)
[ndb_mgmd(MGM)] 1 node(s)
id=1    @192.168.0.3  (Version: 4.1.9)
[mysqld(API)]   2 node(s)
id=4   (Version: 4.1.9)
id=5   (Version: 4.1.9)
ndb_mgm>
```
如果上面没有问题，现在开始测试MySQL：  
注意，这篇文档对于MySQL并没有设置root密码，推荐你自己设置Server1和Server2的MySQL root密码。  
在Server1中：  
```console
/usr/local/mysql/bin/mysql -u root -p

> use test;
> CREATE TABLE ctest (i INT) ENGINE=NDBCLUSTER;
> INSERT INTO ctest () VALUES (1);
> SELECT * FROM ctest;
```
应该可以看到1 row returned信息（返回数值1）。  
如果上述正常，则换到Server2上重复上面的测试，观察效果。如果成功，则在Server2中执行INSERT再换回到Server1观察是否工作正常。  
如果都没有问题，那么恭喜成功！  

###### 破坏性测试  
将Server1或Server2的网线拔掉，观察另外一台集群服务器工作是否正常（可以使用SELECT查询测试）。测试完毕后，重新插入网线即可。  
如果你接触不到物理服务器，也就是说不能拔掉网线，那也可以这样测试：  
在Server1或Server2上：  
```console
# ps aux | grep ndbd
```
将会看到所有ndbd进程信息  
```console
root      5578  0.0  0.3  6220 1964 ?        S    03:14   0:00 ndbd
root      5579  0.0 20.4 492072 102828 ?     R    03:14   0:04 ndbd
root     23532  0.0  0.1  3680  684 pts/1    S    07:59   0:00 grep ndbd
```
然后杀掉一个ndbd进程以达到破坏MySQL集群服务器的目的：  
```console
# kill -9 5578 5579
```
之后在另一台集群服务器上使用SELECT查询测试。并且在管理节点服务器的管理终端中执行show命令会看到被破坏的那台服务器的状态。
测试完成后，只需要重新启动被破坏服务器的ndbd进程即可：  
```console
# ndbd
```
注意！前面说过了，此时是不用加--inital参数的！  
至此，MySQL集群就配置完成了