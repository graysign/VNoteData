# BIND服务器所有域名解析到一个IP
创建zone文件  
```bash
# vim /var/named/chroot/var/named/test.com.zone
$TTL 86400
@ IN SOA @ root (
42 ; serial (d. adams)
3H ; refresh
15M ; retry
1W ; expiry
1D ) ; minimum

IN NS @
IN A 127.0.0.1
IN AAAA ::1
* IN A IP地址

```
