注： 可用 ixfr+nsupdate+rndc 更新域  
```bash
    rndc reconfig
    rndc reload
```
nsupdate是一个动态DNS更新工具.可以向DNS服务器提交更新记录的请求.它可以从区文件中添加或删除资源记录,而不需要手动进行编辑区文件.  
下面是使用方法：  
```bash
nsupdate [ -d ] [ [ -y keyname:secret ] [ -k keyfile ] ] [ -v ] [ filename ]
    -d 调试模式.
    -k 从keyfile文件中读取密钥信息.
    -y keyname是密钥的名称,secret是以base64编码的密钥.
    -v 使用TCP协议进行nsupdate.默认是使用UDP协议.
```
输入格式:  
nsupdate可以从终端或文件中读取命令.每个命令一行.一个空行或一个”send”命令,则会将先前输入的命令发送到DNS服务器上.  
命令格式:  
`server servername [ port ]`
发送请求到servername服务器的port端口.如果不指定servername,nsupdate将把请求发送给当前去的主DNS服务器.如:  
```bash
>server 192.168.0.1 53
local address [ port ]    #发送nsupdate请求时,使用的本地地址和端口.
zone zonename    #指定需要更新的区名.
class classname    #指定默认类别.默认的类别是IN.
key name secret    #指定所有更新使用的密钥.
prereq nxdomain domain-name    #要求domain-name中不存在任何资源记录.
prereq yxdomain domain-name    #要求domain-name存在,并且至少包含有一条记录.
prereq nxrrset domain-name [ class ] type    #要求domain-name中没有指定类别的资源记录.
prereq yxrrset domain-name [ class ] type    #要求存在一条指定的资源记录.类别和domain-name必须存在.
update delete domain-name [ ttl ] [ class ] [ type [ data... ] ]    #删除domain-name的资源记录.如果指定了type和data,仅删除匹配的记录.
update add domain-name ttl [ class ] type data…    #添加一条资源记录.
show    #显示自send命令后,所有的要求信息和更新请求.
send    #将要求信息和更新请求发送到DNS服务器.等同于输入一个空行.
```
nsupdate示例:
```bash
# nsupdate
> server 127.0.0.1
> update delete www.centos.bz A
>
> update add www.centos.bz 80000 IN A 192.168.0.2
> update add 2.0.168.192.in-addr.arpa 80000 PTR A www.centos.bz
> send
> quit
```
LAMP+bind9下实现DDNS生成文件，并调用命令nsuodate -v filename来实现ddns。  

* lamp安装    这个就不用多说了，我安装的是ubuntu的6.0.6.1server+mysql+php    apt-get到最新版
* webmin     俺linux刚刚入门，好多命令行还不熟。服务器不安装gui，用用webmin算是gui吧。
* php-cli    没有安装？用apt-get 安装。  
* bind9及配置 用webmin配置bind
   * 创建新的主区域（zone）比如cags.org.cn
   * 修改named.conf.local 在zone cags.org.cn添加这么一行：`allow-update {127.0.0.1;};` 意思是这个zone在本地可以update。
* mysql建表 用5个字段实现最基本的功能 
    id 自增类型，当主键也当dns的序号用  
    user 用户名  
    pass 密码  
    time 更新的时间  
    ip 客户端的ip
 sql语句如下：  
 ```sql
 CREATE TABLE `ddns` ( 
    `id` int(11) unsigned NOT NULL auto_increment, 
    `user` char(4) NOT NULL, 
    `password` char(41) NOT NULL, 
    `time` bigint(20) NOT NULL default '0', 
    `ip` char(15) NOT NULL, 
    PRIMARY KEY (`id`), 
    UNIQUE KEY `user` (`user`), 
    KEY `password` (`password`), 
    KEY `time` (`time`), 
    KEY `ip` (`ip`) 
) ENGINE=MyISAM DEFAULT
 ```
* php网页文件ddns.php(这个就不用注释了吧，跟大白话似的)
```php
<?php
$user= $_REQUEST["user"];
$pass= $_REQUEST["pass"];
$time= time();
$ip = $_SERVER["REMOTE_ADDR"];

$link = @mysql_connect('localhost', 'root', '');
@mysql_select_db('ddns');

$query="select count(user) from ddns where user='";
$query.=$user;
$query.="' and password=password('";
$query.=$pass;
$query.="') group by user";


$result=@mysql_query($query);
$row = mysql_fetch_array($result,MYSQL_NUM);
if ($row[0]==1){
$query="update ddns set time=$time,ip='$ip' where user='$user'";
@mysql_query($query);
}
@mysql_free_result($result);
@mysql_close($link);
?>
```
建立完文件之后用http://ddns-server-ip/ddns.php?user=XXX&pass=YYY就可以获取客户端ip地址并记录到ddns表中，并且记下了客户端访问的时间。  

* php-cli文件
    * 所需要的记录都在ddns表中。在这里规定每隔60秒更新一次。
    * 如果60秒客户端没有刷新的话，就更改其ip为127.0.0.1
    * 生成nsupdate所需的文件。文件包括两部分：
        * 失效的用户即ip为127.0.0.1的
        *  有效的用户：更新时间在60秒之内并且ip不为127.0.0.1的
    * 调用nsupdate命令 文件如下update.php：
```php
#!/usr/bin/php
<?php
$time=time();
$link = @mysql_connect('localhost', 'root', '');
@mysql_select_db('ddns');

$handle = fopen("/tmp/file.txt", "w");
$str="server 127.0.0.1\r\n";
$str.="zone cags.org.cn\r\n";
@fwrite($handle,$str);

$query="select user,id,ip from ddns where $time-time>0 and $time-time<=60";
$result=@mysql_query($query);
while($row = @mysql_fetch_array($result,MYSQL_NUM)){
    $str="update delete $row[0].cags.org.cn A\r\n";
    $str.="update add $row[0].cags.org.cn $row[1] IN A $row[2] \r\n";
    @fwrite($handle, $str);
}

$query="update ddns set ip='127.0.0.1' where $time-time>60";
@mysql_query($query);

$query="select user,id,ip from ddns where ip='127.0.0.1'";
$result=@mysql_query($query);
while($row = @mysql_fetch_array($result,MYSQL_NUM)){
    $str="update delete $row[0].cags.org.cn A\r\n";
    $str.="update add $row[0].cags.org.cn $row[1] IN A $row[2] \r\n";
    @fwrite($handle, $str);
} 

$str="send \r\n";
@fwrite($handle,$str);
@fclose($handle);
exec('nsupdate -v /tmp/file.txt');

@mysql_free_result($result);
@mysql_close($link);
?>
```

* 定时执行文件php-cli 用webmin 添加corn 任务调度每分钟执行一次
* 用dig命令查看更新是否有效： `dig @127.0.0.1 cags.org.cn axfr`
另一调用示例：  
```bash
nsupdate -v << EOF
server 127.0.0.1
zone test.com
update delete test2.test.com
update delete test3.test.com
send
EOF
nsupdate -v << EOF
server 127.0.0.1
zone test.com
update add test2.test.com 30 IN A 127.0.0.2
update add test3.test.com 30 IN A 127.0.0.3
show
send
EOF
```

