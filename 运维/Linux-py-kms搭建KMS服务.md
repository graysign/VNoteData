# py-kms搭建KMS服务
众所周知，KMS激活方式是当前广大网民“试用”windows，office的最广泛的激活方式。几乎可以用于微软的全线产品。  
但是在本机使用KMS类的激活工具总是有些不放心，一方面每隔180天都要重新激活，另外是这些工具来源不一，经常被杀软查杀，这些激活工具到底有没有安全问题？只能全靠信仰了。  
因此，当前最能体现技术宅们不折腾不死心的做法就是在非本机环境下搭建kms激活模拟器，对局域网内机器进行远程激活。  

py-kms发布地址： https://github.com/myanaloglife/py-kms  
* 安装依赖    `yum install python-argparse`  
* 下载代码    `git clone https://github.com/myanaloglife/py-kms`  
* 运行激活服务器    
```bash
cd /py-kms
python server.py
```
这时候看到提示消息
`TCP server listening at 0.0.0.0 on port 1688.`  
就是说KMS服务已经在1688端口上打开了，没有错误。这就搭建完毕了。  
* 手动激活office 2013    
我的office 是32位的2010版本，所以首先打开有管理员权限的命令行工具，进入程序安装目录：  
```bash
CD "%ProgramFiles(x86)%\MICROSOFT OFFICE\OFFICE14"
```
运行激活命令：

```bash
CSCRIPT OSPP.VBS /SETHST:192.168.0.xxx:1688
CSCRIPT OSPP.VBS /ACT
CSCRIPT OSPP.VBS /DSTATUS
```
以上三行的大意是：  
1.设置激活服务器地址为192.168.0.xxx:1688，即你的内网kms服务器地址；  
2.激活；  
3.查看激活状态。  

