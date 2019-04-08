# 用iptables来防止web服务器被CC攻击
当apache站点受到严重的cc攻击，我们可以用iptables来防止web服务器被CC攻击，实现自动屏蔽IP的功能。  
* 系统要求  
    * LINUX 内核版本：2.6.9-42ELsmp或2.6.9-55ELsmp（其它内核版本需要重新编译内核，比较麻烦，但是也是可以实现的）。  
    * iptables版本：1.3.7  
* 安装
    * 安装iptables1.3.7和系统内核版本对应的内核模块kernel-smp-modules-connlimit
* 配置相应的iptables规则 示例如下：
    * 控制单个IP的最大并发连接数
    ```bash
    iptables -I INPUT -p tcp --dport 80 -m connlimit --connlimit-above 50 -j REJECT #允许单个IP的最大连接数为 30
    ```
    * 控制单个IP在一定的时间（比如60秒）内允许新建立的连接数  
    ```bash
    iptables -A INPUT -p tcp --dport 80 -m recent --name BAD_HTTP_ACCESS --update --seconds 60 --hitcount 30 -j REJECT
    iptables -A INPUT -p tcp --dport 80 -m recent --name BAD_HTTP_ACCESS --set -j ACCEPT
    #单个IP在60秒内只允许最多新建30个连接
    ```
* 验证
    * 工具：flood_connect.c（用来模拟攻击)
    * 查看效果： 
    ```bash
    watch 'netstat -an | grep:21 | \ grep<模拟攻击客户机的IP>| wc -l'
    ```
    * 实时查看模拟攻击客户机建立起来的连接数  
    ```bash
    watch 'iptables -L -n -v | \grep<模拟攻击客户机的IP>'
    ```
    查看模拟攻击客户机被 DROP 的数据包数。  
* 注意 为了增强iptables防止CC攻击的能力，最好调整一下ipt_recent的参数如下：  
```bash
cat /etc/modprobe.conf
options ipt_recent ip_list_tot=1000 ip_pkt_list_tot=60    #记录1000个IP地址，每个地址记录60个数据包
#modprobe ipt_recent
```