mysql 5.6 bin-log双主配置  

##### 环境：  
master1:    192.168.169.101  
端口:         3307  
备注:        由于主机上安装了多个实例，采用mysqld_multi管理（该部分与主从复制无关）  

master2:    192.168.169.102  
端口:          3307  

##### 步骤
1. 确保master1及master2机器mysql实例已安装好
2. 启动双主数据库，创建同步用户  

master 1
```mysql
mysqld_multi start 1
GRANT REPLICATION SLAVE ON *.* to 'syc'@'192.168.169.102' identified by 'syc';
flush privileges;
```
master 2
```mysql
service mysqld start
GRANT REPLICATION SLAVE ON *.* to 'syc'@'192.168.169.101' identified by 'syc';
flush privileges;
```

3. my.cnf配置    

master1配置文件  
```ini
[mysqld1]
basedir = /mysql
datadir = /mysqldata
port = 3307
socket = /tmp/mysql.sock
log-bin=mysql-bin
server-id=1
auto-increment-increment = 2
auto-increment-offset = 1
log-slave-updates
slave-skip-errors=all
sync_binlog=1
innodb_data_file_path=ibdata1:76M;ibdata2:12m:autoextend
```

master2配置文件  
```ini
basedir = /mysql
datadir = /mysqldata
port = 3307
log-bin=mysql-bin
server-id=2
auto-increment-increment = 2
auto-increment-offset = 2
log-slave-updates
slave-skip-errors=all
sync_binlog=1
```

4. 记录bin-log和pos位置，master1和master2分别记录  

master1  
```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

master2
```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000006 |      410 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

5. 启动复制  

master1上  
```mysql
stop slave
change master to master_host='192.168.169.102',master_user='syc',
master_password='syc',master_port=3307,
master_log_file='mysql-bin.000006',master_log_pos=410;
start slave
```

master2上  
```mysql
stop slave
change master to master_host='192.168.169.101',master_user='syc',
master_password='syc',master_port=3307,
master_log_file='mysql-bin.000005',master_log_pos=120;
start slave
```

6. 检查状态  

master1和master2都检查  
```mysql
show slave status\G;

Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

7. 建表测试  

###### 参数说明  
```mysql
auto-increment-increment= 2 # 应设为整个结构中服务器的总数
auto-increment-offset = 1   # 设定数据库中自动增长的起点，避免两台服务器数据同步时出现主键冲突
log-slave-updates           # 这个参数用来配置从服务器的更新是否写入二进制日志，这个选项默认是不打开的，但是，如果这个从服务器B是服务器A的从服务器，同时还作为服务器C的主服务器，那么就需要开发这个选项，这样它的从服务器C才能获得它的二进制日志进行同步操作
--slave-skip-errors=[err1,err2,…….|ALL]           # 在复制过程中，由于各种的原因，从服务器可能会遇到执行BINLOG中的SQL出错的情况，在默认情况下，服务器会停止复制进程，不再进行同步，等到用户自行来处理。当复制过程中遇到定义的错误号，就可以自动跳过，直接执行后面的SQL语句.但必须注意的是，启动这个参数，如果处理不当，很可能造成主从数据库的数据不同步，在应用中需要根据实际情况，如果对数据完整性要求不是很严格，那么这个选项确实可以减轻维护的成本
```

sync_binlog: 这个参数是对于MySQL系统来说是至关重要的，他不仅影响到Binlog对MySQL所带来的性能损耗，而且还影响到MySQL中数据的完整性。对于`sync_binlog`参数的各种设置的说明如下：
```mysql
sync_binlog=0，当事务提交之后，MySQL不做fsync之类的磁盘同步指令刷新binlog_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘。
sync_binlog=n，当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘。
```
在MySQL中系统默认的设置是sync_binlog=0，也就是不做任何强制性的磁盘刷新指令，这时候的性能是最好的，但是风险也是最大的。因为一旦系统Crash，在binlog_cache中的所有binlog信息都会被丢失。而当设置为“1”的时候，是最安全但是性能损耗最大的设置。因为当设置为1的时候，即使系统Crash，也最多丢失binlog_cache中未完成的一个事务，对实际数据没有任何实质性影响。从以往经验和相关测试来看，对于高并发事务的系统来说，“sync_binlog”设置为0和设置为1的系统写入性能差距可能高达5倍甚至更多。  

