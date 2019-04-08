# CentOS下PHP安装SSH2扩展
首先，为PHP安装SSH2扩展需要两个软件包，libssh2和ssh2。两者的最新版本分别为1.4.2和0.12，下载地址分别为  
http://www.libssh2.org/download/和http://pecl.php.net/package/ssh2。  
这里我们可以均下载最新版本，libssh2的源码包为libssh2-1.4.2.tar.gz，ssh2的源码包为ssh2-0.12.tgz。  
其次，解压并安装libssh2和ssh2。其中，libssh2需要先安装，ssh2后安装。安装步骤如下：  
```bach
tar -zxvf libssh2-1.4.2.tar.gz
cd libssh2-1.4.2
./configure --prefix=/usr/local/libssh2
make && make install
```
以上为安装libssh2，这里需要记住libssh2的安装目录，因为在安装ssh2的时候还会用到。  
```bash
tar -zxvf ssh2-0.12.tgz
cd ssh2-0.12
phpize
./configure --prefix=/usr/local/ssh2 --with-ssh2=/usr/local/libssh2
make
```
执行完以上过程后，在当前目录下的modules目录下会生成一个ssh2.so文件，这就是扩展PHP所需要的，将该文件拷贝到PHP库的存储目录下在修改PHP的配置文件即可。  
```bash
cp modules/ssh2.so /usr/lib64/php/modules/
```
注：PHP库的存储目录可能因系统而异，本博主的机器上是`/usr/lib64/php/modules/`
```bash
vi /etc/php.ini
# 向该文件中添加内容：
extension=ssh2.so
```
此时为PHP扩展SSH2就已经完成了，为了验证是否安装成功，我们可以通过执行一下命令来验证。  
```bash
php -i|grep ssh2
Registered PHP Streams => php, file, http, ftp, compress.bzip2, compress.zlib, https, ftps, ssh2.shell, ssh2.exec, ssh2.tunnel, ssh2.scp, ssh2.sftp  
ssh2
libssh2 version => 1.4.2
banner => SSH-2.0-libssh2_1.4.2
```
最后，我们再通过一个简单的PHP程序来试用SSH2，该程序首先连接远程服务器，然后执行相关操作，最后读取操作执行的返回结果，具体例子代码如下。
```php
<?php
    $user="user";
    $pass="password";
    $connection=ssh2_connect('202.112.113.250',22);
    ssh2_auth_password($connection,$user,$pass);
    $cmd="ps aux";
    $ret=ssh2_exec($connection,$cmd);
    stream_set_blocking($ret, true);
    echo (stream_get_contents($ret));
?>
```