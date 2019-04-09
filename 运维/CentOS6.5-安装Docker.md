# CentOS6.5安装Docker
```bash
yum --enablerepo=elrepo-kernel install kernel-ml -y
yum --enablerepo=elrepo-kernel install kernel-lt -y
```
问题1： docker: relocation error: docker: symbol dm_task_get_info_with_deferred_remove, version Base not defined in file libdevmapper.so.1.02 with link time reference    
解决办法:`yum upgrade device-mapper-libs`    

```bash
[root@localhost ~] # uname -r
2.6.32-431.el6.x86_64

[root@localhost ~] # cat /etc/issue
CentOS release 6.5 (Final) 	Kernel \r on an \m

注意其他的源可能导致你的内核和docker的版本不一致，需要升级内核至3.x。
[root@localhost ~] # rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
Retrieving http: //dl .fedoraproject.org /pub/epel/6/x86_64/epel-release-6-8 .noarch.rp
warning:  /var/tmp/rpm-tmp .JN76fI: Header V3 RSA /SHA256  Signature, key ID 0608b895: NOKEY  
Preparing...              ########################################### [100%]
1:epel-release            ########################################### [100%]
[root@localhost ~] # rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
[root@localhost ~] # yum -y install docker-io
```
启动并设置开机自动启动  

```bash
[root@localhost ~] # service docker start
Starting cgconfig service:                                 [确定]
Starting docker:                                      [确定]
[root@localhost ~] # chkconfig docker on
```
获取cnetos镜像  
```bash
[root@localhost ~] # docker pull centos:latest
centos:latest: The image you are pulling has been verified
511136ea3c5a: Pull complete
5b12ef8fd570: Pull complete
34943839435d: Downloading [===>                                               ] 18.38 MB /232 .5 MB 1h7m49s
```
官方安装方式docker pull imagename从docker的索引中心下载，imagename是镜像名称，例如docker pull ubuntu就是下载base ubuntu并且tag是latest。  
我们还可以搜索基于 Fedora 和 Ubuntu 操作系统的容器。  
```bash
[root@localhost ~] # docker search ubuntu
[root@localhost ~] # docker search fedora
```
查看docker镜像  
```bash
[root@localhost ~] # docker images centos
REPOSITORY          TAG                 IMAGE ID            CREATED                  VIRTUAL SIZE 															centos              latest              34943839435d        Less than a second ago   224 MB
```
运行docker运行shell  
```bash
[root@localhost ~] # docker run -i -t centos /bin/bash
```
停止容器
```bash
[root@localhost ~] # docker stop <CONTAINER ID>
```
删除所有容器
```bash
docker  rm  $(docker  ps  -a -q)
```
查看docker的子命令，直接敲docker 或完整的docker help 就可以  
常用命令  
总结一下常用命令:  
其中<>阔起来的参数为必选，[]阔起来为可选  
docker version 查看docker的版本号，包括客户端、服务端、依赖的Go等  
docker info 查看系统(docker)层面信息，包括管理的images, containers数等
docker search <image> 在docker index中搜索image  
docker pull <image> 从docker registry server 中下拉image  
docker push <image|repository> 推送一个image或repository到registry  
docker push <image|repository>:TAG 同上，指定tag  
docker inspect <image|container> 查看image或container的底层信息  
docker images TODO filter out the intermediate image layers (intermediate image layers 是什么)  
docker images -a 列出所有的images  
docker ps 默认显示正在运行中的container  
docker ps -l 显示最后一次创建的container，包括未运行的  
docker ps -a 显示所有的container，包括未运行的  
docker logs <container> 查看container的日志，也就是执行命令的一些输出  
docker rm <container...> 删除一个或多个container  
```bash
docker rm `docker ps -a -q` 删除所有的container
```
docker ps -a -q | xargs docker rm 同上, 删除所有的container  
docker rmi <image...> 删除一个或多个image  
docker start/stop/restart <container> 开启/停止/重启container  
docker start -i <container> 启动一个container并进入交互模式  
docker attach <container> attach一个运行中的container  
docker run <image> <command> 使用image创建container并执行相应命令，然后停止  
docker run -i -t <image> /bin/bash 使用image创建container并进入交互模式, login shell是/bin/bash  
docker run -i -t -p <host_port:contain_port> 将container的端口映射到宿主机的端口  
docker commit <container> [repo:tag] 将一个container固化为一个新的image，后面的repo:tag可选  
docker build <path> 寻找path路径下名为的Dockerfile的配置文件，使用此配置生成新的image  
docker build -t repo[:tag] 同上，可以指定repo和可选的tag  
docker build - < <dockerfile> 使用指定的dockerfile配置文件，docker以stdin方式获取内容，使用此配置生成新的image  
docker port <container> <container port> 查看本地哪个端口映射到container的指定端口，其实用docker ps 也可以看到  


