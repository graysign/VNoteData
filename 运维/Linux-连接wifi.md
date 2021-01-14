```console
yum install  wireless-tools.x86_64
#(1)首先关闭开发板的有线网卡
[root@FriendlyARM /]# ifconfig eth0 down
#(2)加载USB WiFi无线网卡
[root@FriendlyARM /]# ifconfig wlan0 up
#(3)扫描可用的无线网络
[root@FriendlyARM /]# iwlist scanning | grep ESSID
lo        Interface doesn't support scanning.
eth0      Interface doesn't support scanning.
wmaster0  Interface doesn't support scanning.
                    ESSID:"FRIENDLY-ARM"
                    ESSID:"NETGEAR"
                    ESSID:"TP-LINK"
#(4)选择要连接的无线网络
[root@FriendlyARM /]# iwconfig wlan0 essid "FRIENDLY-ARM"
#(5)输入该网络的安全密码
[root@FriendlyARM /]# iwconfig wlan0 key s:12345
#(6)连接到指定的AP(无线路由)
[root@FriendlyARM /]# iwconfig wlan0 ap auto
#(7)设置无线网卡的IP地址
[root@FriendlyARM /]# ifconfig wlan0 192.168.1.120
#(8)使用 ping 命令检测无线网连通状况
[root@FriendlyARM /]# ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1): 56 data bytes
64 bytes from 192.168.1.1: seq=0 ttl=64 time=42.804 ms
64 bytes from 192.168.1.1: seq=1 ttl=64 time=5.020 ms
```

```console
1、iwlist 命令：用于对/proc/net/wireless文件进行分析，得出无线网卡相关信息
# iwlist wlan0 scanning 搜索当前无线网络
# iwlist wlan0 frequen  显示频道信息
# iwlist wlan0 rate  显示连接速度
# iwlist wlan0 power  显示电源模式
# iwlist wlan0 txpower 显示功耗
# iwlist wlan0 retry  显示重试连接次数(网络不稳定查看)
# iwlist wlan0 ap 显示热点信息
# iwlist --help 显示帮助信息
# iwlist --version 显示版本信息

2、iwconfig  系统配置无线网络设备或显示无线网络设备信息。iwconfig 命令类似于ifconfig命令，但是他配置对象是无线网卡，它对网络设备进行无线操作，如设置无线通信频段
auto 自动模式
essid 设置ESSID
nwid 设置网络ID
freq 设置无线网络通信频段
chanel 设置无线网络通信频段
sens 设置无线网络设备的感知阀值
mode 设置无线网络设备的通信设备
ap 强迫无线网卡向给定地址的接入点注册
nick<名字> 为网卡设定别名
rate<速率> 设定无线网卡的速率
rts<阀值> 在传输数据包之前增加一次握手，确信信道在正常的
```