###### 环境  
 mysql 版本： 5.6.11  
 操作系统版本: rhel 6.2  

Master 的 my.cnf 配置( 只贴M/M 结构部分）  
```ini
log-bin=fabian
server-id=1
binlog-do-db=TSC
binlog-do-db=adb
binlog-ignore-db=mysql
replicate-do-db=TSC
replicate-do-db=adb
replicate-ignore-db=mysql
log-slave-updates
slave-skip-errors=all
auto_increment_increment=2
auto_increment_offset=1
```

Slave 的my.cnf 配置(只贴M/M结构部分)  
```ini
log-bin=fabian
server-id=2
binlog-do-db=adb
binlog-ignore-db=mysql
replicate-do-db=TSC
replicate-do-db=adb
replicate-ignore-db=mysql
log-slave-updates
slave-skip-errors=all
auto_increment_increment=2
auto_increment_offset=2
```

对上面参数作出部分解释：  
```ini
log-bin=fabian                             #M/S 需开启log-bin 日记文件
server-id=1                                #指定server-id 必须不一致，M/s 结构时 M > S
binlog-do-db=TSC                           #同步数据库名称
binlog-ignore-db=mysql                     #忽略数据名称
replicate-do-db=TSC                        #用于控制slave来执行同步的行为
replicate-ignore-db=mysql                  #用于控制slave来执行同步的行为
log-slave-updates                          #把更新的记录写到二进制文件中
slave-skip-errors=all                      #跳过错误，继续执行复制
auto_increment_increment=2                 #设置主键单次增量
auto_increment_offset=1                    #设置单次增量中主键的偏移量
#expire_logs_days = 20                     #设置log-bin 超过多少天删除
max-binlog-size= 512M
# auto_increment_increment、auto_increment_offset 可以防止双主主键冲突问题
```

对开启权限、grant 这些基本的这里就不在详细说明，面贴出自动建立同步的gant 脚本，项目在生产过程中总会遇到Mysql 数据库服务器宕机等情况，可用以下脚本来重新构建Master -to-Master 环境。  

```bash
#!/bin/bash
# Setting Variables
_REMOTEHOST=192.168.1.51  #远程主机IP
_LOCALHOST=192.168.1.52   #本地主机IP
_USER=root                #用户名
_REMOTEPASD=123456        #远程主机密码
_LOCALPASD=123456         #本地主机密码
_BASE=TSC
_LF=`mysql -u root -h $_REMOTEHOST -p$_REMOTEPASD -e "show master status\G;" | awk '/File/ {print $2}'`
_LLF=`mysql -u root -p$_LOCALPASD -e "show master status\G;" | awk '/File/ {print $2}'`
_PS=`mysql -u root -h $_REMOTEHOST -p$_REMOTEPASD -e "show master status\G;" | awk '/Position/ {print $2}'`
_LPS=`mysql -u root -p$_LOCALPASD -e "show master status\G;" | awk '/Position/ {print $2}'`

# Backup Mysql
mysqldump -u root -h $_REMOTEHOST -p$_REMOTEPASD  $_BASE > $_BASE.sql
mysql -u root -p$_LOCALPASD $_BASE < $_BASE.sql
rm -rf $_BASE.sql
mysql -uroot -p$_LOCALPASD -e "stop slave;"
mysql -h $_REMOTEHOST -uroot -p$_LOCALPASD -e "stop slave;"

echo "mysql -uroot -p$_LOCALPASD -e +change master to master_REMOTEHOST=*${_REMOTEHOST}*,master_user=*${_USER}*,master_password=*${_REMOTEPASD}*,master_log_file=*${_LF}*,master_log_pos=${_PS};+" > tmp

echo "mysql -h $_REMOTEHOST -uroot -p$_LOCALPASD -e +change master to master_REMOTEHOST=*${_LOCALHOST}*,master_user=*${_USER}*,master_password=*${_LOCALPASD}*,master_log_file=*${_LLF}*,master_log_pos=${_LPS};+" > tmp2

sed -ri 's/\+/"/g' tmp
sed -ri 's/\+/"/g' tmp2
sed -ri "s/\*/\'/g" tmp
sed -ri "s/\*/\'/g" tmp2
sh tmp
sh tmp2
rm -rf tmp
rm -rf tmp2

mysql -uroot -p$_LOCALPASD -e "start slave;"
mysql -h $_REMOTEHOST -uroot -p$_LOCALPASD -e "start slave;"
mysql -uroot -p$_LOCALPASD -e "show slave status\G;" | awk '$0 ~/Host/ || $0 ~/State/'
mysql -h $_REMOTEHOST -uroot -p$_LOCALPASD -e "show slave status\G;" | awk '$0 ~/Host/ || $0 ~/State/'

```

#脚本执行完成后出现下图则表示成功：  
```bash
[root@ORA2 fabian]# sh gant.sh 
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.

Slave_IO_State: Queueing master event to the relay log
Master_Host: 192.168.1.51
Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it

Warning: Using a password on the command line interface can be insecure.
Slave_IO_State: Queueing master event to the relay log
Master_Host: 192.168.1.52
Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
[root@ORA2 fabian]#

```

测试 master-to-master 是否成功要用 命令行界面或都三方工具(如 Navicat Premium) 进行对同步数据库进行测试，看是否在另一库进行同步操作。  
