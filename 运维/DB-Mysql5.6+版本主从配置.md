###### mysql5.5+版本与mysql5.5之前版本的一些差异：  
```ini
master-host=192.168.124.5   #5.5之后不再支持
master-user= AffairLog      #5.5之后不再支持
master-password= password   #5.5之后不再支持
master-port=3306            #5.5之后不再支持
master-connect-retry=60     #5.5之后不再支持

server-id=1
log-bin=log
binlog-do-db=database1 //需要同步的数据库
binlog-do-db=database2 
binlog-ignore-db=mysql //被忽略的数据库 

//replicate_do_db和replicate_ignore_db时有一个隐患，跨库更新时会出错
//https://blog.csdn.net/yu757371316/article/details/65633220
replicate-do-db=database1 //同步的数据库 
replicate-do-db=database2 
replicate-ignore-db=mysql //被忽略的数据库
```

```mysql
change master to master_host='116.121.1.10',master_port=1223,master_user='newback',master_password='cctv@12315#$',master_log_file='mysql-bin.000001',master_log_pos=120
slave start
```

###### 准备  
* 首先确保mysql主从服务器之间的数据库端口防火墙互相打开，尽量确保主从数据库账户一致性(主从切换使用)，否则将操作失败，
* 其次是确保mysql账户对mysql数据库目录有“可读写”权限非“可写”权限，为了确保不出意外，最好删除mysql之前陈旧的mysql-bin、mysql日志，然后重启mysql  

###### 主从服务器分别作以下操作：  
* 版本一致
* 初始化表，并在后台启动mysql
* 修改root的密码  
  
###### 修改主服务器master:  
```console
#vi /etc/my.cnf
```
```ini
[mysqld]
log-bin=mysql-bin //[必须]启用二进制日志
server-id=1 //[必须]默认是1，一般取IP最后一段
port=1223
bing-address=0.0.0.0

log-bin=mysql-bin.log（必须，数据库日志文件，主从必须）
binlog-do-db =new_test (要记录的数据库,多个可换行多次设置)
replicate-do-db =new_test (要复制的数据库，多个可换行过个设置)

binlog-ignore-db=mysql //不对mysql库进行日志记录操作 如下意思雷同
binlog-ignore-db=test
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema

replicate-ignore-db=test //不对test进行复制操作 如下意思雷同
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
```

###### 修改从服务器slave:  
```console
#vi /etc/my.cnf
```
```ini
[mysqld]
port=1223
bing-address=0.0.0.0 //意思是允许所有 机器 服务器安全起见可设置为指定的服务器IP地址 如 116.128.1.10等
log-bin=mysql-bin.log
server-id=2
binlog-do-db =new_test
replicate-do-db =new_test

binlog-ignore-db=mysql
binlog-ignore-db=test
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema

replicate-ignore-db=test
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
```

###### 重启两台服务器的mysql  
```console
service mysql restart
```

###### 在主服务器上建立帐户并授权slave:  
```console
#/usr/local/mysql/bin/mysql -u数据库账户名 -p数据库密码
```
```mysql
mysql>GRANT REPLICATION SLAVE ON *.* to 'newback_username'@'%' identified by 'newback_pwd'; //一般不用root帐号，“%”表示所有客户端都可能连（安全起见可将%替换成指定服务器IP,如116.121.1.10），只要帐号，密码正确。
```

###### 登录主服务器的mysql，查询master的状态（可在phpmyadmin 中执行次操作）  
```mysql
mysql>show master status;
+------------------+----------+--------------+------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 | 120 | | |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```
注：`执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化`  

###### 配置从服务器Slave：  
```mysql
mysql>change master to master_host='116.121.1.10',master_port=1223,master_user='newback',master_password='cctv@12315#$',master_log_file='mysql-bin.000001',master_log_pos=120 //注意不要断开，master_port为mysql服务器端口号(无引号)，master_user为执行同步操作的数据库账户，“120”无单引号(此处的120就是show master status 中看到的position的值，这里的mysql-bin.000001就是file对应的值)。(此处可在从服务器phpmyadmin中用sql语句操作)

Mysql>start slave; //启动从服务器复制功能（可在phpmyadmin中执行该SQL语句）
```

###### 检查从服务器复制功能状态：  
```mysql
mysql> show slave status\G (可在从服务器phpmyadmin中执行“show slave status” SQL语句)
*************************** 1. row ***************************

……………………(省略部分)
Slave_IO_Running: Yes //此状态必须YES
Slave_SQL_Running: Yes //此状态必须YES
……………………(省略部分)
```
注：`Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。`
以上操作过程，主从服务器配置完成。  

###### 主从服务器测试：  
在主服务器new_test数据库中的test表中插入 或者更新一条记录，如果从服务器同样更新 插入 则配置正确，否则错误  
