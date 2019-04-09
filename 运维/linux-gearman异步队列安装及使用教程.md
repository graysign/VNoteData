性能   

* 以下测试数值为虚拟机   
* 常规HTTP的Curl：这个受端口最大值及打开文件句柄数限制 
* 当使用HTTP Curl时 
* 同步：2000/sec 

*  Gearman：未优化情况下 
* 用PHP+gearman揑件测试并发批量调用： 
* 同步：4000/sec 
* 异步：10000/sec 
* 以上还有上升空间，官方测试数据为5w/sec 
* 增加持久化揑件设置后性能会下降一些  

##### gearman异步队列安装流程  

###### 基础安装包 
```bash
yum install vim wget gcc gcc-c++ make dos2unix gperf libevent libevent-devel zlib-devel bzip2-devel openssl-devel ncurses-devel    boost boost-devel mysql-devel
```
###### 安装gearman 异步队列
```bash
wget https://launchpad.net/gearmand/1.2/1.1.9/+download/gearmand-1.1.9.tar.gz
tar -zxvf gearmand-1.1.9.tar.gz
cd gearmand-1.1.9
./configure   如果出现错误请查看下面的错误解决
```
成功后如下   
```bash
* LIBS:                      
* LDFLAGS Flags:             
* Assertions enabled:        no
* Debug enabled:             no
* Warnings as failure:       no
* Building with libsqlite3   no
* Building with libdrizzle   no
* Building with libmemcached not found
* Building with libpq        no
* Building with tokyocabinet no
* Building with libmysql     yes
* SSL enabled:               no
* make -j:                   3
* VCS checkout:              no
```
```bash
make
make install
```

###### 安装gearman php 扩展  
```bash
wget http://pecl.php.net/get/gearman
mv gearman gearman.tar.gz
tar -zxvf gearman.tar.gz
cd gearman-1.1.2/
phpize
./configure
make
make install


cd /etc/php.d/
cp gd.ini gearman.ini
vim gearman.ini

; Enable gearman extension module
extension=gearman.so


service php-fpm restart
```

错误解决  

* 在configure过程中出现了以下错误： 
```bash
checking for Boost headers version >= 1.39.0… no
configure: error: cannot find Boost headers version >= 1.39.0
```
解决办法：  
```bash
yum search boost
yum install boost.x86_64
yum install boost-devel.x86_64
```

* 执行./configure出现以下错误   
```bash
checking for gperf... no
configure: error: could not find gperf
```
解决办法：   
```bash
yum search gperf
yum install gperf.x86_64
```

* 执行./configure出现以下错误   
```bash
checking test for a working libevent... no
configure: error: Unable to find libevent
```
解决办法：   
```bash
yum install libevent libevent-devel
```

gearman 参数说明  
```bash
Client mode: gearman [options] [<data>]
Worker mode: gearman -w [options] [<command> [<args> ...]]

Common options to both client and worker modes.
    -f <function> - Function name to use for jobs (can give many)
    -h <host>     - Job server host
    -H            - Print this help menu
    -v            - Print diagnostic information to stdout(false)
    -p <port>     - Job server port
    -t <timeout>  - Timeout in milliseconds
    -i <pidfile>  - Create a pidfile for the process

Client options:
    -b            - Run jobs in the background(false)
    -I            - Run jobs as high priority
    -L            - Run jobs as low priority
    -n            - Run one job per line(false)
    -N            - Same as -n, but strip off the newline(false)
    -P            - Prefix all output lines with functions names
    -s            - Send job without reading from standard input
    -u <unique>   - Unique key to use for job

Worker options:
    -c <count>    - Number of jobs for worker to run before exiting
    -n            - Send data packet for each line(false)
    -N            - Same as -n, but strip off the newline(false)
    -w            - Run in worker mode(false) 
```

##### gearman异步队列使用：  
下面先做个命令行测试：  
首先开两个命令行窗口：  
tty1:
```bash
gearman -w -f abc  -- wc  -m
```
表示统计用户输入了多少个字符。  

tty2:  
```bash
gearman -f abc 'aaaa'
```  


tty1输出结果:`4`输出结果正确。  

可以直接从文件中读入内容：  
```bash
gearman -f abc < /etc/php.ini
```

###### 使用php做同步列队测试：  
```bash
vi /var/www/html/company/gearman/worker.php
```
```php
<?php
$worker= new GearmanWorker();
$worker->addServer('127.0.0.1', 4730); //连接job服务器
$worker->addFunction('reverse', 'my_reverse_function');  //注册支持任务及对应函数
 
while ($worker->work());    //循环等待任务，没有时阻塞，循环体内可以放错误处理
//work内尽量不要出现资源忘记回收情况
//处理任务的回调函数
 
function my_reverse_function($job)
{
    $workload = $job->workload();   //过来的参数
    $result = strrev($workload);    //运算
    $content = file_get_contents("http://www.google.com");    //加大运算时间
    
    //加入执行日志
    $file = fopen("worker_counter.log","a+");
    fwrite($file,date("Y-m-d H:i:s")."\n");
    //fwrite($file,var_export($job,TRUE)."\n");
    fwrite($file,$workload."\n");
    fwrite($file,$result."\n");
    fclose($file);
    
    return $result;                    //返回给调用方的数据
}
?>
```

```bash
vi /var/www/html/company/gearman/client.php
```
```php

<?php
try{
    $client= new GearmanClient();
    $client->addServer('127.0.0.1', 4730);
    echo $client->do('reverse', 'You can do it.'), "\n";
}catch(Exception $e){
    print_r($e);
}
?>
```  

使用tty1运行worker  
```bash
php worker.php
```
使用tty2运行client  
```bash
php client.php
.ti od nac uoY
```
输出结果正确  

使用浏览器运行client  
```bash
http://192.168.252.128/company/gearman/client.php
.ti od nac uoY 
```
输出结果正确  


###### 下面使用php做异步列队测试：  
```bash
vi /var/www/html/company/gearman/worker.php
```  
```php

<?php
$worker= new GearmanWorker();
$worker->addServer('127.0.0.1', 4730);
$worker->addFunction('reverse', 'my_reverse_function');
 
while ($worker->work());
 
function my_reverse_function($job)
{
    $workload = $job->workload();
    $result = strrev($workload);
    $content = file_get_contents("http://www.google.com");    //加大运算时间
    
    //加入执行日志
    $file = fopen("worker_counter.log","a+");
    fwrite($file,date("Y-m-d H:i:s")."\n");
    //fwrite($file,var_export($job,TRUE)."\n");
    fwrite($file,$workload."\n");
    fwrite($file,$result."\n");
    fclose($file);
    
    return $result;
}
?>
```

```bash
vi /var/www/html/company/gearman/client.php
```
```php
<?php
try{
    $client= new GearmanClient();
    $client->addServer('127.0.0.1', 4730);
    echo $client->doBackground('reverse', 'You can do it.'), "\n";    //异步只是派发任务，不等待返回结果
}catch(Exception $e){
    print_r($e);
}
?>
```


使用tty1运行worker:  
```bash
php worker.php
```

tty2查看执行效果:  
```bash
tail -f /var/www/html/company/gearman/worker_counter.log
```

tty3运行client:  
```bash
php client.php
H:localhost:150
```

表示该操作列队中，我可以在tty2中实时查看到效果  
```bash
2014-03-15 06:08:16
You can do it.
.ti od nac uoY
```

使用浏览器运行client:  
```bash
http://192.168.252.128/company/gearman/client.php
H:localhost:153 
```

表示该操作列队中，我可以在tty2中实时查看到效果  
```bash
2014-03-15 06:09:44
You can do it.
.ti od nac uoY
```

我们看到了client不管执行任何操作，都会立即得到一个列队ID，剩余的操作交给了worker。  

