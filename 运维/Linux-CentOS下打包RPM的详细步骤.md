首先，我们需要了解一些预备知识，在旧版本的CentOS中，rpmbuild工具默认的工作路径是`/usr/src/redhat`，因为权限原因，一般用户身份不能制作rpm软件包，只能切换到root身份才能够制作。在新版本的CentOS中，可以在一般用户主目录下新建rpmbuild目录作为rpmbuild工具的工作目录。并且，在新版本中，发行商建议为了防止系统函数库或其他文件损坏，不要使用root身份去制作rpm软件包。  

#### 详细步骤  

##### 必要工具的安装  

其中包括make工具，若你的待安装包使用C语言写的，还需要安装gcc编译程序。最重要的是要安装rpmbuild工具。命令如下：  

```console
[root@hostname ~]# yum install make
[root@hostname ~]# yum install gcc
[root@hostname ~]# yum install rpmbuild
```
##### 创建制作rpm的工作目录  

为了创建制作rpm的工作目录，你需要在一般用户身份的主目录下新建如下结构的目录：  
rpmbuild目录，还有该目录下六个目录，包括：BUILD、BUILDROOT、RPMS、SOURCES、SPECS、SRPMS，命令如下：  

```console
[userid@hostname ~]$ mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}

#各个目录的一般用途如下简介
#BUILD 	    编译rpm包的临时目录 	
#BUILDROOT   编译后生成的软件临时安装目录
#RPMS 	    最终生成的可安装rpm包的所在目录
#SOURCES     所有源代码和补丁文件的存放目录
#SPECS 	    存放SPEC文件的目录(重要)
#SRPMS 	    软件最终的rpm源码格式存放路径
```

##### 制作源代码文件tarball生成  

这里主要利用《鸟哥的LINUX私房菜》中提供的源代码进行演示。将自己的源代码打包，其中可以包括可执行文件、脚本文件、用户使用手册、配置文件还有一些其他文件。然后将打包好的压缩包放在SOURCES目录下。  

```console
[userid@hostname ~]$ tar -zcv -f main-0.l.tar.gz main-0.1        #假设主目录下存在待打包的包括源代码的目录=>main-0.1
[userid@hostname ~]$ cp main-0.1.tar.gz ~/rpmbuild/SOURCES       #将打包好的源代码拷贝到SOURCES目录下
```

##### 新建*.spec的设置文件  

这个spec文件的设置是整个打包过程中最重要的一环，必须仔细设置。在SPECS目录下新建该目录，并且进入该文件进行设置，其中文件中各个宏名（例如%install）的具体含义请参考文献【4】  

```console
[userid@hostname ~]$ cd ~/rpmbuild/SPECS
[userid@hostname ~]$ vim main.spec
Name:           main
Version:        0.1
Release:        1%{?dist}
Summary:        Calculate sin and cos value

License:        GPL
URL:            http://linux.vbird.org
Source0:        main-0.1.tar.gz


%description
This package will let you input your name and calculate sin and cos value

%prep
%setup -q


%build                                      #执行编译命令，编译后会在BUILD目录下存在暂时文件
make

%install                     
rm -rf %{buildroot}
mkdir -p %{buildroot}/usr/local/bin         #将编译完成源代码试安装在~/rpmbuild/BUILDROOT目录下，其中的宏%{buildroot}=~/rpmbuild
make install RPM_INSTALL_ROOT=%{buildroot}

%files                                      #最终在安装生成的rpm软件包时的安装目录
/usr/local/bin/main


%changelog
* Wed Jul 4 2018 VBird Tsai <vbird@mail.vbird.idv.tw> 0.1
- Build the program 
```

##### 编译成为RPM和SRPM  
输入如下命令，在最后出现exit 0表示生成成功。  

```console
[userid@hostname ~]$ rpmbuild -ba main.spec
```

##### 安装和测试  
输入如下命令进行安装和测试。  

```console

[userid@hostname ~]$ rpm -ivh ~/rpmbuild/RPMS/x86_64/main-0.1-1.el7.x86_64.rpm    #安装main
[userid@hostname ~]$ rpm -ql main                                                 #查找main的安装路径
[userid@hostname ~]$ rpm -qi main                                                 #查询main相关信息
```

#### 注意事项  
在/usr/local/bin目录中不能有同名目录或者文件存在  

#### 参考文献  
【1】https://www.cnblogs.com/masterpanda/p/5700453.html  

【2】https://access.redhat.com/sites/default/files/attachments/rpm_building_howto.pdf  

【3】https://wiki.centos.org/HowTos/SetupRpmBuildEnvironment  

【4】https://docs.fedoraproject.org/quick-docs/en-US/creating-rpm-packages.html#con_rpm_spec_file_overview