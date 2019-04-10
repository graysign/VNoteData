###### 常用命令示例
1. 命令 -A, --append  
范例：`iptables -A INPUT -p tcp --dport 80 -j ACCEPT`  
说明 : 新增规则到INPUT规则链中，规则时接到所有目的端口为80的数据包的流入连接，该规则将会成为规则链中的最后一条规则。  

2. 命令 -D, --delete  
范例：`iptables -D INPUT -p tcp --dport 80 -j ACCEPT` 或 `iptables -D INPUT 1`  
说明： 从INPUT规则链中删除上面建立的规则，可输入完整规则，或直接指定规则编号加以删除。  

3. 命令 -R, --replace  
范例： `iptables -R INPUT 1 -s 192.168.0.1 -j DROP`
说明 取代现行第一条规则，规则被取代后并不会改变顺序。  

4. 命令 -I, --insert  
范例：`iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT`  
说明：在第一条规则前插入一条规则，原本该位置上的规则将会往后移动一个顺位。  

5. 命令 -L, --list  
范例： `iptables -L INPUT`  
说明：列出INPUT规则链中的所有规则。  

6. 命令 -F, --flush  
范例： `iptables -F INPUT`  
说明： 删除INPUT规则链中的所有规则。  

7. 命令 -Z, --zeroLINUX教程  centos教程  
范例：`iptables -Z INPUT`  
说明 将INPUT链中的数据包计数器归零。它是计算同一数据包出现次数，过滤阻断式攻击不可少的工具。  

8. 命令 -N, --new-chain  
范例：` iptables -N denied`  
说明： 定义新的规则链。  

9. 命令 -X, --delete-chain  
范例： `iptables -X denied`  
说明： 删除某个规则链  

10. 命令 -P, --policy  
范例 ：`iptables -P INPUT DROP`  
说明 ：定义默认的过滤策略。 数据包没有找到符合的策略，则根据此预设方式处理。  

11. 命令 -E, --rename-chain  
范例： `iptables -E denied disallowed`  
说明： 修改某自订规则链的名称。  



###### 常用封包比对参数：  
1. 参数 -p, --protocol  
范例：`iptables -A INPUT -p tcpLINUX教程  centos教程`  
说明：比对通讯协议类型是否相符，可以使用 ! 运算子进行反向比对，例如：-p ! tcp ，意思是指除 tcp 以外的其它类型，包含udp、icmp ...等。如果要比对所有类型，则可以使用 all 关键词，例如：-p all。  

2. 参数 -s, --src, --source  
范例: `iptables -A INPUT -s 192.168.1.100`  
说明：用来比对数据包的来源IP，可以比对单机或网络，比对网络时请用数字来表示屏蔽，例如：-s 192.168.0.0/24，比对 IP 时可以使用!运算子进行反向比对，例如：-s ! 192.168.0.0/24。  

3. 参数 -d, --dst, --destination  
范例：` iptables -A INPUT -d 192.168.1.100`  
说明：用来比对封包的目的地 IP，设定方式同上。  

4. 参数 -i, --in-interface  
范例 `iptables -A INPUT -i  lo`  
说明:用来比对数据包是从哪个网卡进入，可以使用通配字符 + 来做大范围比对，如：-i eth+ 表示所有的 ethernet 网卡，也可以使用 ! 运算子进行反向比对，如：-i ! eth0。这里lo指本地换回接口。  

5. 参数 -o, --out-interface  
范例：`iptables -A FORWARD -o eth0`  
说明：用来比对数据包要从哪个网卡流出，设定方式同上。  

6. 参数 --sport, --source-port  
范例：`iptables -A INPUT -p tcp --sport 22`
说明：用来比对数据的包的来源端口号，可以比对单一端口，或是一个范围，例如：--sport 22:80，表示从 22 到 80 端口之间都算是符合件，如果要比对不连续的多个端口，则必须使用 --multiport 参数，详见后文。比对端口号时，可以使用 ! 运算子进行反向比对。    

7. 参数 --dport, --destination-port  
范例 : `iptables -A INPUT -p tcp --dport 22`  
说明:  用来比对封包的目的地端口号，设定方式同上。    

8. 参数 --tcp-flags  
范例：`iptables -p tcp --tcp-flags SYN,FIN,ACK SYN  `
说明：比对 TCP 封包的状态标志号，参数分为两个部分，第一个部分列举出想比对的标志号，第二部分则列举前述标志号中哪些有被设，未被列举的标志号必须是空的。TCP 状态标志号包括：SYN（同步）、ACK（应答）、FIN（结束）、RST（重设）、URG（紧急）PSH（强迫推送） 等均可使用于参数中，除此之外还可以使用关键词 ALL 和 NONE 进行比对。比对标志号时，可以使用 ! 运算子行反向比对。  

9. 参数 --syn  
范例：`iptables -p tcp --syn`  
说明：用来比对是否为要求联机之TCP 封包，与 iptables -p tcp --tcp-flags SYN,FIN,ACK SYN 的作用完全相同，如果使用 !运算子，可用来比对非要求联机封包。  

10. 参数 -m multiport --source-port  
范例： `iptables -A INPUT -p tcp -m multiport --source-port 22,53,80,110 -j ACCEPT`
说明: 用来比对不连续的多个来源端口号，一次最多可以比对 15 个端口，可以使用 ! 运算子进行反向比对。  

11. 参数 -m multiport --destination-port  
范例 ：`iptables -A INPUT -p tcp -m multiport --destination-port 22,53,80,110 -j ACCEPT`  
说明：用来比对不连续的多个目的地端口号，设定方式同上。    

12. 参数 -m multiport --port  
范例：`iptables -A INPUT -p tcp -m multiport --port 22,53,80,110 -j ACCEPT`  
说明：这个参数比较特殊，用来比对来源端口号和目的端口号相同的数据包，设定方式同上。注意：在本范例中，如果来源端口号为 80，目的地端口号为 110，这种数据包并不算符合条件。  

13. 参数 --icmp-type  
范例：`iptables -A INPUT -p icmp --icmp-type 8 -j DROP`
说明：用来比对 ICMP 的类型编号，可以使用代码或数字编号来进行比对。请打 iptables -p icmp --help 来查看有哪些代码可用。这里是指禁止ping如，但是可以从该主机ping出。  

14. 参数 -m limit --limit  
范例：`iptables -A INPUT -m limit --limit 3/hour`  
说明：用来比对某段时间内数据包的平均流量，上面的例子是用来比对：每小时平均流量是否超过一次3个数据包。 除了每小时平均次外，也可以每秒钟、每分钟或每天平均一次，默认值为每小时平均一次，参数如后： /second、 /minute、/day。 除了进行数据包数量的比对外，设定这个参数也会在条件达成时，暂停数据包的比对动作，以避免因洪水攻击法，导致服务被阻断。  

15. 参数 --limit-burst  
范例：`iptables -A INPUT -m limit --limit-burst 5`  
说明：用来比对瞬间大量封包的数量，上面的例子是用来比对一次同时涌入的封包是否超过 5 个（这是默认值），超过此上限的封将被直接丢弃。使用效果同上。  

16. 参数 -m mac --mac-source  
范例：`iptables -A INPUT -m mac --mac-source 00:00:00:00:00:01 -j ACCEPT`  
说明：用来比对数据包来源网络接口的硬件地址，这个参数不能用在 OUTPUT 和 Postrouting 规则链上，这是因为封包要送出到网后，才能由网卡驱动程序透过 ARP 通讯协议查出目的地的 MAC 地址，所以 iptables 在进行封包比对时，并不知道封包会送到个网络接口去。linux基础  

17. 参数 --mark  
范例：`iptables -t mangle -A INPUT -m mark --mark 1`  
说明：用来比对封包是否被表示某个号码，当封包被比对成功时，我们可以透过 MARK 处理动作，将该封包标示一个号码，号码最不可以超过 4294967296。linux基础  

18. 参数 -m owner --uid-owner  
范例：`iptables -A OUTPUT -m owner --uid-owner 500`
说明：用来比对来自本机的封包，是否为某特定使用者所产生的，这样可以避免服务器使用 root 或其它身分将敏感数据传送出，可以降低系统被骇的损失。可惜这个功能无法比对出来自其它主机的封包。  

19. 参数 -m owner --gid-owner  
范例：`iptables -A OUTPUT -m owner --gid-owner 0`  
说明：用来比对来自本机的数据包，是否为某特定使用者群组所产生的，使用时机同上。  

20. 参数 -m owner --pid-owner  
范例：`iptables -A OUTPUT -m owner --pid-owner 78`  
说明：用来比对来自本机的数据包，是否为某特定行程所产生的，使用时机同上。  

21. 参数 -m owner --sid-owner  
范例： `iptables -A OUTPUT -m owner --sid-owner 100`  
说明： 用来比对来自本机的数据包，是否为某特定联机（Session ID）的响应封包，使用时机同上。  

22. 参数 -m state --state  
范例： `iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT`  
说明 用来比对联机状态，联机状态共有四种：INVALID、ESTABLISHED、NEW 和 RELATED。

23. iptables -L -n -v  可以查看计数器  
INVALID 表示该数据包的联机编号（Session ID）无法辨识或编号不正确。ESTABLISHED 表示该数据包属于某个已经建立的联机。NEW 表示该数据包想要起始一个联机（重设联机或将联机重导向）。RELATED 表示该数据包是属于某个已经建立的联机，所建立的新联机。例如：FTP-DATA 联机必定是源自某个 FTP 联机。  


###### 常用的处理动作  
-j 参数用来指定要进行的处理动作，常用的处理动作包括：`ACCEPT、REJECT、DROP、REDIRECT、MASQUERADE、LOG、DNAT、SNAT、MIRROR、QUEUE、RETURN、MARK。`  
分别说明如下：  

* `ACCEPT `    将数据包放行，进行完此处理动作后，将不再比对其它规则，直接跳往下一个规则链（natostrouting）。
*  `REJECT `    拦阻该数据包，并传送数据包通知对方，可以传送的数据包有几个选择：ICMP port-unreachable、ICMP echo-reply 或是tcp-reset（这个数据包会要求对方关闭联机），进行完此处理动作后，将不再比对其它规则，直接 中断过滤程序。 范例如下：  
    ```bash
    iptables -A FORWARD -p TCP --dport 22 -j REJECT --reject-with tcp-reset
    ```
* `DROP `    丢弃包不予处理，进行完此处理动作后，将不再比对其它规则，直接中断过滤程序。
*  `REDIRECT `    将包重新导向到另一个端口（PNAT），进行完此处理动作后，将会继续比对其它规则。 这个功能可以用来实作通透式porxy 或用来保护 web 服务器。例如：  
    ```bash
    iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
    ```
* `MASQUERADE `     改写数据包来源 IP为防火墙 NIC IP，可以指定 port 对应的范围，进行完此处理动作后，直接跳往下一个规则（mangleostrouting）。这个功能与 SNAT 略有不同，当进行 IP 伪装时，不需指定要伪装成哪个 IP，IP 会从网卡直接读取，当使用拨号连接时，IP 通常是由 ISP 公司的 DHCP 服务器指派的，这个时候 MASQUERADE 特别有用。范例如下：linux基础  
```bash
iptables -t nat -A POSTROUTING -p TCP -j MASQUERADE --to-ports 1024-31000
```

* `LOG `    将封包相关讯息纪录在 /var/log 中，详细位置请查阅 /etc/syslog.conf 配置文件，进行完此处理动作后，将会继续比对其规则。例如：  
```bash
iptables -A INPUT -p tcp -j LOG --log-prefix "INPUT packets"
```
* `SNAT `    改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则（mangleostrouting）。范例如下：  
```bash
iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000
```
* `DNAT`    改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规炼（filter:input 或 filter:forward）。范例如下：  
```bash
iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination
192.168.1.1-192.168.1.10:80-100
```
* `MIRROR`    镜像数据包，也就是将来源 IP 与目的地 IP 对调后，将数据包送回，进行完此处理动作后，将会中断过滤程序。  
* `QUEUE`    中断过滤程序，将数据包放入队列，交给其它程序处理。透过自行开发的处理程序，可以进行其它应用，例如：计算联机费.......等。  
* `RETURN `    结束在目前规则链中的过滤程序，返回主规则链继续过滤，如果把自订规则链看成是一个子程序，那么这个动作，就相当提早结束子程序并返回到主程序中。
*  `MARK`    将数据包标上某个代号，以便提供作为后续过滤的条件判断依据，进行完此处理动作后，将会继续比对其它规则。范例如下：  
```bash
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j MARK --set-mark 2
```