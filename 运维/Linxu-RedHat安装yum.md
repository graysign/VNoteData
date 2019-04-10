```bash
wget http://yum.baseurl.org/download/2.0/yum-2.0.8-1.src.rpm
rpmbuild --rebuild yum-2.0.8-1.src.rpm
cd /usr/src/redhat/RPMS/noarch
rpm -ivh yum-2.0.8-1.noarch.rpm
```

配置 `/etc/yum.conf `使用兼容的更新源  
```config
[main]
cachedir=/var/cache/yum
debuglevel=2
logfile=/var/log/yum.log
pkgpolicy=newest
distroverpkg=redhat-release
tolerant=1
exactarch=1

[base]
name=CentOS-$releasever - Base
baseurl=http://vault.centos.org/4.9/os/x86_64/
gpgcheck=1

[updates]
name=Red Hat Linux $releasever - Updates
baseurl=http://vault.centos.org/4.9/updates/x86_64/
gpgcheck=1
```

安装CentOS的GPG Key  
```bash
 rpm --import http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-4
```

测试yum是否正常(下面是更新所有的rpm)  
```bash
yum update
#注意: 此方式更新所有已经安装的rpm，你不需要则可以取消
```
