###### 依赖:  
```bash
yum -y install automake autoconf libtool ncurses-devel libxslt groff pcre-devel pkgconfig 
```

###### 编译：  
```bash
wget -c https://repo.varnish-cache.org/source/varnish-4.0.0-beta1.tar.gz
tar -xvf varnish-4.0.0-beta1.tar.gz 
cd varnish-4.0.0-beta1.tar.gz
./autogen.sh 
export PKG_CONFIG=/usr/local/bin/pkg-config
./configure --prefix=/usr/local/varnish
make && make install 
```
安装目录是在`/usr/local/varnish`  
如果报错的话，即pcre问题  
```bash
../../lib/libvarnishapi/.libs/libvarnishapi.so: undefined reference to `pcre_free_study'
collect2: ld returned 1 exit status
make[3]: *** [varnishadm] Error 1 
```
试着将`pcre-devel`删除` yum erase pcre-devel `  
再手动编译pcre  


###### Varnish启动  
```bash
cp redhat/varnish.initrc /etc/init.d/varnish
cp redhat/varnish.sysconfig /etc/sysconfig/varnish
cp redhat/varnish_reload_vcl /usr/bin/varnish_reload_vcl
chmod 755 /etc/init.d/varnish
chkconfig varnish on 
```

一个测试的启动脚本
```bash
/usr/local/varnish/sbin/varnishd -n /var/vcache -f /usr/local/varnish/vcl.conf -a 0.0.0.0:80 -s file,/var/vcache/varnish_cache.data,1G -g www -u www -w 30000,51200,10 -T 127.0.0.1:3500 -p client_http11=on 
```

###### Varnish 语法检测  
```bash
 varnishd -C -f /etc/varnish/default.vcl 
 
```
找到出错的地方了  
```bash
Message from VCC-compiler:
Symbol not found: 'req.request' (expected type BOOL):
('input' Line 39 Pos 9) if (req.request == "PURGE") {
--------###########-------------- Running VCC-compiler failed, exit 1 VCL compilation failed 
```  

###### REDHAT Varnish  
如果不是64位的话，可以这样子  

```bash
rpm --nosignature -i https://repo.varnish-cache.org/redhat/varnish-4.0.el6.rpm yum install varnish
```