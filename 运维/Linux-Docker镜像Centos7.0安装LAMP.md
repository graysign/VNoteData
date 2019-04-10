去`https://registry.hub.docker.com/`搜索对应 TAG：
```bash
docker pull  centos:centos7
```  

以 CentOS 镜像作为基础镜像，启动容器并在其中执行` /bin/bash` 命令，` -t -i `参数用于创建一个容器  
```bash
docker run -t -i centos:centos7 /bin/bash
```

在容器中，执行下面的命令：  
```bash
yum update
yum install httpd httpd-devel
```

安装完成后，输入 exit 退出容器的命令行。  
执行 `docker ps -a`，可以看到被终止的容器

把所做的改变提交到一个新的容器：  
```bash
docker commit [container-id] custom/centos_httpd
```
容器成功提交后，执行 `docker images` ，会看到刚才提交的容器

删除旧容器：  
```bash
docker rm [container-id]
```

删除旧镜像：  
```bash
docker rmi [image-id]
docker rmi [image-name][:image-tag]
```

以刚才提交的容器为基础容器，再来创建一个新的容器：  
```bash
docker run -t -i custom/centos_httpd /bin/bash
apachectl -k start
```
查看 Docker 中 CentOS 系统IP地址：`ip addr`  
查看测试页面（为上一步看到的IP地址）：`http://172.17.0.8/`

```bash
yum install wget tar gcc gcc-c++ make
```

安装PHP 5.4：  
```bash
yum install php php-devel
yum install php-bcmath php-gd libjpeg* php-intl php-ldap php-mbstring php-mhash php- mysql nd php- mysql i php-odbc php-pdo php-pear php-pecl-memcache php-soap php-xml php-xmlrpc  

pecl install apc  

vi /etc/php.ini  
添加：extension=apc.so
```

安装libmcrypt：  
```bash
wget http://sourceforge.net/projects/mcrypt/files/Libmcrypt/2.5.8/libmcrypt-2.5.8.tar.gz
tar -xvzf libmcrypt-2.5.8.tar.gz
cd libmcrypt-2.5.8
./configure
make && make install
```

安装mhash：  
```bash
wget http://sourceforge.net/projects/mhash/files/mhash/0.9.9.9/mhash-0.9.9.9.tar.gz
tar -xvzf mhash-0.9.9.9.tar.gz
cd mhash-0.9.9.9
./configure
make && make install
```

安装mcrypt：  
```bash
wget http://sourceforge.net/projects/mcrypt/files/MCrypt/2.6.8/mcrypt-2.6.8.tar.gz
tar -xvzf mcrypt-2.6.8.tar.gz
cd mcrypt-2.6.8
LD_LIBRARY_PATH=/usr/local/lib ./configure
make && make install
```

安装php-mcrypt：  
```bash
wget http://museum.php.net/php5/php-5.4.16.tar.gz
tar -xvzf php-5.4.16.tar.gz
cd php-5.4.16/ext/mcrypt
phpize
php-config
./configure
make && make install
vi /etc/php.ini
添加：extension=mcrypt.so

apachectl -k stop
apachectl -k start
```

安装 MySQL Client：  
```bash
rpm -ivh http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
#上面的地址来自 http://dev.mysql.com/downloads/repo/yum/ ，可能会有变动，请自行修改
yum install mysql
```

`ctrl + p  ctrl + q 可退出并使容器继续运行`

安装 MySQL：  
```bash
docker pull mysql:5.6.20
```
启动：  
```bash
docker run --name docker_mysql -e MYSQL_ROOT_PASSWORD=mysecretpassword -d -P mysql:5.6.20
```

查看端口映射：  
```bash
docker ps -a
0.0.0.0:49154->3306/tcp
```
此时容器的3306端口会被映射到宿主机器的49154端口，这样我们就可以通过宿主机器的49154端口访问 docker_mysql 了  
```bash
mysql --host=127.0.0.1 --port=49154 -uroot -pmysecretpassword
```

安装 OpenSSH：  
```bash
yum install openssh-server openssh-clients
ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
#启动 ssh 服务：
/usr/sbin/sshd
#修改密码：
yum install passwd
passwd
#登录：
ssh root@localhost
```

解决 vi 中文乱码：  
```bash
vi /etc/virc
#在最后加上：
set fileencodings=utf-8,gb2312,gbk,gb18030
set termencoding=utf-8
set encoding=prc
```