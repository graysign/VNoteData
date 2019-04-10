```bash
yum -y update
yum -y install gcc gcc-c++ authconf automake libtool libevent libevent-devel
yum -y install ncurse nucrse-devel gd gd-deevel freetype freetype-devel fontconfig fontconfig-devel libjpeg libjpeg-devel zlib zlib-devel pcre pcre-devel
yum -y install libmcrypt mhash
yum -y install mysql mysql-server mysql-devel
yum -y install httpd httpd-devel
yum -y install php*



/etc/init.d/httpd restart    #启动apache
/etc/init.d/mysqld restart    #启动mysql
```

编辑一个php测试文件  
```bash
vi /var/www/html/info.php
```
```php
<?php
phpinfo();
?>
```
现在可以通过`http://ip/info.php`查看信息了  
设置开机启动
```bash
chkconfig httpd on
chkconfig mysqld on
```

附：如果需要修改的话，那么修改配置文件即可。  
附1：mysql配置文件所在位置：  
```bash
# ls -l /etc/my.cnf
-rw-r--r-- 1 root root 441 Nov  4 02:53 /etc/my.cnf
#
```

附2：apache配置文件所在位置:  
```bash
# ls -l /etc/httpd/
total 8
drwxr-xr-x 2 root root 4096 Jan 11 07:06 conf
drwxr-xr-x 2 root root 4096 Jan 11 07:03 conf.d
lrwxrwxrwx 1 root root   19 Jan 11 06:53 logs -> ../../var/log/httpd
lrwxrwxrwx 1 root root   27 Jan 11 06:53 modules -> ../../usr/lib/httpd/modules
lrwxrwxrwx 1 root root   13 Jan 11 06:53 run -> ../../var/run
#
```

附3：php配置文件所在位置：  
```bash
# ls -l /etc/php.ini
-rw-r--r-- 1 root root 45079 Nov 30 00:53 /etc/php.ini
#
```