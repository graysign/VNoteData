## 1.下载编译kernel   

```bash
#web https://www.kernel.org/
mkdir kernel
cd kernel
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.329.tar.xz
tar -Jxf linux-4.9.329.tar.xz
ln -s linux-4.9.329 linux
cd linux
make menuconfig
make -j8 bzImage

cp arch/x86/boot/bzImage ../
```

## 2.构建 initramfs（BusyBox)  
initramfs 是一种以 cpio 格式压缩的根文件系统，它通常和 Linux 内核文件 vmlinuz 一起被打包成 boot.img 作为启动镜像
```bash
#web https://www.busybox.net/

wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2
tar -jxf busybox-1.35.0.tar.bz2
cd busybox-1.35.0
make defconfig
make menuconfig
#选择settings --> Build BusyBox as a static binary (no shared libs)
#save
make -j8
make -j8 install

cd ./_install
mkdir dev 
sudo mknod dev/console c 5 1
sudo mknod dev/ram b 1 0 
touch init
chmod +x init

#create init file
vi init
#!/bin/sh
echo "INIT SCRIPT"
mkdir /proc
mkdir /sys
mount -t proc none /proc
mount -t sysfs none /sys
mkdir /tmp
mount -t tmpfs none /tmp
mknod -m 666 /dev/ttyS0 c 4 64
echo -e "\nThis boot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
setsid cttyhack /bin/sh
exec /bin/sh

#create initramfs.cpio.gz
find . -print0 | cpio --null -ov --format=newc | gzip -9 >  ../initramfs.cpio.gz

```

## 3.下载QEMU虚拟机

```bash
#web https://www.qemu.org/
#1.command install
#QEMU is packaged by most Linux distributions:
#Arch: pacman -S qemu
#Debian/Ubuntu: apt-get install qemu
#Fedora: dnf install @virtualization
#Gentoo: emerge --ask app-emulation/qemu
#RHEL/CentOS: yum install qemu-kvm
#SUSE: zypper install qemu

#2.src build
wget https://download.qemu.org/qemu-7.1.0.tar.xz
tar xvJf qemu-7.1.0.tar.xz
cd qemu-7.1.0
./configure
make

#windows
wget https://qemu.weilnetz.de/w64/qemu-w64-setup-20220831.exe
```


## 4.运行系统

```bash

#-net nic -net tap,ifname=tap0 桥接tap0 网卡
qemu-system-x86_64   -kernel ./bzImage  -initrd  ./initramfs.cpio.gz --append "root=/dev/ram init=/init"  -net nic -net tap,ifname=tap0

#qemu-system-x86_64 -s  -kernel ./bzImage  -initrd  ./initramfs.cpio.gz --append "nokaslr root=/dev/ram init=/init console=ttyS0"  -nographic 

#配置网络
ip addr add 192.168.31.110/24  dev eth0 up
ip link set etho up

rout add defualt gw 192.168.31.1

```




## 5.参考链接

[QEMU搭建Kernel模拟调试开发环境](https://blog.csdn.net/RIPNDIP/article/details/104966711)  
[通过 QEMU 打开学习 Linux kernel 的新世界](https://www.jianshu.com/p/9b68e9ea5849)   
[Windows版本QEMU网络配置](https://blog.csdn.net/jiangwei0512/article/details/122791901)  
[qemu network](https://www.cnblogs.com/memoryart/p/qemu_network.html)  
[ifconfig配置ip](https://zhidao.baidu.com/question/1763096704882905828.html)