# Linux中TCP参数调优
* 修改文件句柄数量限制
```bash
# 查看当前用户允许TCP打开的文件句柄最大数
ulimit -n

# 修改文件句柄
vim /etc/security/limits.conf

* soft nofile 655350
* hard nofile 655350
```
修改后，退出终端窗口，重新登录(不需要重启服务器)，就能看到最新的结果了。这是优化的第一步，修改文件句柄限制。  
注意：  
soft nofile （软限制）是指Linux在当前系统能够承受的范围内进一步限制用户同时打开的文件数
hard nofile （硬限制）是根据系统硬件资源状况(主要是系统内存)计算出来的系统最多可同时打开的文件数量
通常软限制小于或等于硬限制  
* TCP参数调优  

| 参数 | 默认配置 | 调整配置 | 说明 |  
| --- | --- | --- | --- |  
| fs.file-max | 1048576 | 9999999 | 所有进程打开的文件描述符数 |  
| fs.nr_open | 1635590 | 1635590 | 单个进程可分配的最大文件数 |  
| net.core.rmem_default | 124928 | 262144 | 默认的TCP读取缓冲区 |  
| net.core.wmem_default | 124928 | 262144 | 默认的TCP发送缓冲区 |  
| net.core.rmem_max | 124928 | 8388608 | 默认的TCP最大读取缓冲区 |  
| net.core.wmem_max | 124928 | 8388608 | 默认的TCP最大发送缓冲区 |  
| net.ipv4.tcp_wmem | 4096 16384 4194304 | 4096 16384 8388608 | TCP发送缓冲区 |  
| net.ipv4.tcp_rmem | 4096 87380 4194304 | 4096 87380 8388608 | TCP读取缓冲区 |  
| net.ipv4.tcp_mem | 384657 512877 769314 | 384657 512877 3057792 | TCP内存大小 |  
| net.core.netdev_max_backlog | 1000 | 5000 | 在每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目 |  
| net.core.optmem_max | 20480 | 81920 | 每个套接字所允许的最大缓冲区的大小 |  
| net.core.somaxconn | 128 | 2048 | 每一个端口最大的监听队列的长度，这是个全局的参数 |  
| net.ipv4.tcp_fin_timeout | 60 | 30 | 对于本端断开的socket连接，TCP保持在FIN-WAIT-2状态的时间（秒）。对方可能会断开连接或一直不结束连接或不可预料的进程死亡 |  
| net.core.netdev_max_backlog | 1000 | 10000 | 在每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目 |  
| net.ipv4.tcp_max_syn_backlog | 1024 | 2048 | 对于还未获得对方确认的连接请求，可保存在队列中的最大数目。如果服务器经常出现过载，可以尝试增加这个数字 |  
| net.ipv4.tcp_max_tw_buckets | 5000 | 5000 | 系统在同时所处理的最大timewait sockets数目 |  
| net.ipv4.tcp_tw_reuse | 0 | 1 | 是否允许将TIME-WAIT sockets重新用于新的TCP连接 |  
| net.ipv4.tcp_keepalive_time | 7200 | 900 | 表示TCP链接在多少秒之后没有数据报文传输时启动探测报文（发送空的报文） |  
| net.ipv4.tcp_keepalive_intvl | 75 | 30 | 表示前一个探测报文和后一个探测报文之间的时间间隔 |  
| net.ipv4.tcp_keepalive_probes | 9 | 3 | 表示探测的次数 |  

从上面的配置参数中我们可以知道，在Linux内核中为tcp发送和接收都做了缓冲队列，这样可以提高系统的吞吐量。  
以上这些参数都是在 `/etc/sysctl.conf` 文件中定义的，有的参数在文件中可能没有定义，系统给定了默认值，需要修改的话，直接在文件中添加或修改，然后执行  
`sysctl -p`命令让其生效。  
注意：  
参数值并不是设置的越大越好，有的需要考虑服务器的硬件配置，参数对服务器上其它服务的影响等。

