# vlmcsd搭建KMS服务
根据github上的说明，这个工具是用C写的，没有任何依赖，可以直接运行。而且它横跨几乎现在所有的系统平台，如Android, FreeBSD, Solaris, Minix, Mac OS, iOS, Windows。  
相比于另一款工具py-kms需要依赖pyhont2或者python3，可谓是非常干净舒爽。  
###### 安装  
* 在任意环境中，下载最新的vlmcsd releases版本，下载地址。如在linux中，可以使用wget下载：
    `wget https://github.com/Wind4/vlmcsd/releases/download/svn1111/binaries.tar.gz`
* 解压我们下载的包，进入对应的目录。如Ubuntu系统，我们可以进入binaries/Linux/intel/static目录中  
* 选择对应的文件，这里我们选择vlmcsdmulti-x64-musl-static文件。然后把这个文件放到我们想要的文件中  
* 执行chmod命令，为这个文件赋予权限：  
   `chmod u+x /usr/local/KMS-server`  
   权限赋予完毕之后，直接执行命令
   `./vlmcsdmulti-x64-musl-static vlmcsd`  
   如果没有任何错误提示，代表我们成功了。不放心的话，可以再执行一遍，会提示我们端口(1688)和地址已经被占用。  
* 若有防火墙，记得把1688端口开放，然后加入自启动。如在Ubuntu中，可以编辑/etc/rc.local文件，在启动项里加入启动命令。
* 复制以下文本
```bat
cd /d "%SystemRoot%\system32"
slmgr /skms 你的VPS的IP或者域名,域名似乎不能加http之类的
slmgr /ato
slmgr /xpr
```
存成bat格式的文件，然后右键以管理员身份运行。  
* 验证是否激活。在cmd或powershell中执行    `slmgr.vbs -dlv`
   不出意外的话，会显示已经激活成功的信息
* 这个kms激活服务器，同样可以用来激活office，原理基本一致。可以参考原作者的[github pages](http://wind4.github.io/vlmcsd/)进行激活。