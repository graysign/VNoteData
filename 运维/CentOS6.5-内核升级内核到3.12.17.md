环境：  
系统硬件：vmware vsphere (CPU：2*4核，内存2G)  
系统版本：Linux centos 2.6.32-431.el6.x86_64（Centos-6.5-x86_64-minimal.iso ）  
升级内核版本：longterm:3.12.17  

升级步骤：  
1. 虚拟系统安装  
    要求mininal方式安装(205个包),具体步骤省略。  
2. 查看原有系统内核版本，升级更新包  
    
   * 更新包
```bash
[root@centos ~]# yum update
[root@centos ~]# yum upgrade 
```
   * 查看系统内核版本
```bash
[root@centos ~]# uname -a
Linux centos 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
[root@centos ~]# cat /etc/redhat-release
CentOS release 6.5 (Final)
```
3. 下载、安装需编译环境所需要的工具包
```bash
[root@centos ~]# yum install vim wget
[root@centos ~]# yum install gcc gcc-c++ xz
[root@centos ~]# yum install bc
[root@centos ~]# yum install ncurses-devel
[root@centos ~]# yum install hmaccalc zlib-devel binutils-devel elfutils-libelf-devel
[root@centos ~]# yum install qt-devel #如果有X环境时安装(目前不安装)
```

4.  下载内核包  

    * 进行目录  
```bash
[root@centos ~]# cd /usr/local/src/
```
    * 下载内核包，目前是3.12.17版本。
```bash
[root@centos ~]# wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.12.17.tar.xz
```

5. 编译内核  
    * 解包，进行内核源码目录 
    ```bash
    [root@centos ~]# tar -vxf linux-3.12.17.tar.xz
    [root@centos ~]# cd linux-3.12.17
    #/usr/local/src/linux-3.12.17此目录当编译目录，编译过程，操作都必须在此目录
    ```
    * 以菜单的方式，选择编译内核需要的模块
    ```bash
    [root@centos ~]# make menuconfig
    #执行完make menuconfig后，修改/usr/src/linux-2.6.35.4/.config将#CONFIG_SYSFS_DEPRECATED_V2 is not set默认被注释掉的，将其改为y。即修改为CONFIG_SYSFS_DEPRECATED_V2=y
    ```
    * 查看当前版本，并且以原编译配置来进行编译  

    ```bash
    [root@centos ~]# uname -r
    2.6.32-431.el6.x86_64
    ```
    
    * 复制原配置文件到编译目录（视需要，把旧的配合文件做为新的配合模板）  
    
    ```bash
    [root@centos ~]# cp /boot/config-2.6.32-431.11.2.el6.x86_64 .config
    提示是否覆盖，输入Y
    [root@centos ~]# sudo sh -c 'yes "" | make oldconfig'
    以原配置文件产生新的配置文件，默认回答为YES方式
    ```

    * 编译内核 （需时约30-40分钟）  
 
    ```bash
    [root@centos ~]# make
    ```

    * 安装内核  
    
```bash
[root@centos ~]# make modules_install install
完成时，会提示 could not find module vmware_balloon，这个和虚拟机有关（不理它）
```

6. 更改系统启动时，使用的内核  
```bash
[root@centos ~]# vim /boot/grub/menu.lst
修改default=0，开机后，默认以第一项启动（3.12.17内核）
保存退出
```

7. 重启系统  
```bash
[root@centos ~]# shutdown -r now
```

8. 确认当前内核版本  
```bash
[root@centos ~]# uname -r
Linux centos 3.12.17 #1 SMP Fri Apr 11 03:32:42 CST 2014 x86_64 x86_64 x86_64 GNU/Linux
```
显示内核为3.12.17，表示升级内核成功  

9. 如果编译失败，可以先清除，再重新编译  
```bash
[root@centos ~]# cd /usr/local/src/linux-3.12.17
[root@centos ~]# make mrproper         #完成或者安装过程出错，可以清理上次编译的现场
[root@centos ~]# make clean
```

10. 如果升级成功后，可以删除源码目录  
```bash
[root@centos ~]# rm -rf /usr/local/src/linux-3.12.17
```
11. 删除原来的内核  
    * 查看当前有什么内核版本  
```bash
[root@centos ~]# rpm -q kernel
#显示以下版本
kernel-2.6.32-431.el6.x86_64
kernel-2.6.32-431.11.2.el6.x86_64
```
    * 删除原内核  
```bash
[root@centos ~]# yum remove kernel-2.6.32-431.el6.x86_64 #移除此版本的内核，同时启动菜单也不再会出现此内核
[root@centos ~]# yum remove kernel-2.6.32-431.11.2.el6.x86_64
#删除后，查看启动菜单会发现已经少了此内核
[root@centos ~]# cat /boot/grub/menu.lst
```