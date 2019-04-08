###### Linux下查CC攻击
* 查看所有80端口的连接数  
```bash
netstat -nat|grep -i "80"|wc -l
```
* 对连接的IP按连接数量进行排序  
```bash
netstat -anp | grep 'tcp\|udp' | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
netstat -ntu | awk '{print $5}' | egrep -o "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" | sort | uniq -c | sort -nr
```
* 查看TCP连接状态
```bash
netstat -nat |awk '{print $6}'|sort|uniq -c|sort -rn
netstat -n | awk '/^tcp/ {print $NF}'|sort|uniq -c|sort -rn
netstat -n | awk '/^tcp/ {++S[$NF]};END {for(a in S) print a, S[a]}'
netstat -n | awk '/^tcp/ {++state[$NF]}; END {for(key in state) print key,"\t",state[key]}'
netstat -n | awk '/^tcp/ {++arr[$NF]};END {for(k in arr) print k,"\t",arr[k]}'
netstat -ant | awk '{print $NF}' | grep -v '[a-z]' | sort | uniq -c
```
* 查看80端口连接数最多的20个IP
```bash
cat /www/web_logs/waitalone.cn_access.log|awk '{print $1}'|sort|uniq -c|sort -nr|head -100
tail -n 10000 /www/web_logs/waitalone.cn_access.log|awk '{print $1}'|sort|uniq -c|sort -nr|head -100
cat /www/web_logs/waitalone.cn_access.log|awk '{print $1}'|sort|uniq -c|sort -nr|head -100
netstat -anlp|grep 80|grep tcp|awk '{print $5}'|awk -F: '{print $1}'|sort|uniq -c|sort -nr|head -n20
netstat -ant |awk '/:80/{split($5,ip,":");++A[ip[1]]}END{for(i in A) print A,i}' |sort -rn|head -n20
```
* 用tcpdump嗅探80端口的访问看看谁最高
```bash
tcpdump -i eth0 -tnn dst port 80 -c 1000 | awk -F"." '{print $1"."$2"."$3"."$4}' | sort | uniq -c | sort -nr |head -20
```
* 查找较多time_wait连接
```bash
netstat -n|grep TIME_WAIT|awk '{print $5}'|sort|uniq -c|sort -rn|head -n20
```
* 查找较多的SYN连接
```bash
netstat -an | grep SYN | awk '{print $5}' | awk -F: '{print $1}' | sort | uniq -c | sort -nr | more
```

###### iptables封ip段的一些常见命令  
* 封单个IP的命令是
```bash
iptables -I INPUT -s 211.1.0.0 -j DROP
```
* 封IP段的命令是
```bash
iptables -I INPUT -s 211.1.0.0/16 -j DROP
iptables -I INPUT -s 211.2.0.0/16 -j DROP
iptables -I INPUT -s 211.3.0.0/16 -j DROP
```
* 封整个段的命令是
```bash
iptables -I INPUT -s 211.0.0.0/8 -j DROP
```
* 封几个段的命令是
```bash
iptables -I INPUT -s 61.37.80.0/24 -j DROP
iptables -I INPUT -s 61.37.81.0/24 -j DROP
```
* 服务器启动自运行
```bash
1、把它加到/etc/rc.local中
2、iptables-save >/etc/sysconfig/iptables可以把你当前的iptables规则放到/etc/sysconfig/iptables中，系统启动iptables时自动执行。
3、service iptables save 也可以把你当前的iptables规则放/etc/sysconfig/iptables中，系统启动iptables时自动执行。
```
后两种更好此，一般iptables服务会在network服务之前启来，更安全。
* 解封
```bash
iptables -D INPUT -s IP地址 -j REJECT
iptables -F 全清掉了
```
