
#### rpmbuild介绍  
顾名思义创建rpm包，它是用来指示转换的源码不定编译成二进制文件的包，在centos下默认目录为`/usr/src/redhat`  

#### 目录  

```console

--BUILD #编译之前，如解压包后存放的路径
--BUILDROOT #编译后存放的路径
--RPMS #打包完成后rpm包存放的路径
--SOURCES #源包所放置的路径
--SPECS #spec文档放置的路径
--SPRMS #源码rpm包放置的路径

注：一般我们都把源码打包成tar.gz格式然后存放于SOURCES路径下，而在SPECS路径下编写spec文档，通过命令打包后，默认会把打包后的rpm包放在RPMS下，
而源码包会被放置在SRPMS下.
```

#### rpmbuild相关命令  

```console
基本格式：rpmbuild [options] [spec文档|tarball包|源码包]

1.  从spec文档建立有以下选项：
    -bp  #只执行spec的%pre 段(解开源码包并打补丁，即只做准备)
    -bc  #执行spec的%pre和%build 段(准备并编译)
    -bi  #执行spec中%pre，%build与%install(准备，编译并安装)
    -bl  #检查spec中的%file段(查看文件是否齐全)
    -ba  #建立源码与二进制包(常用)
    -bb  #只建立二进制包(常用)
    -bs  #只建立源码包
2.  从tarball包建立，与spec类似
    -tp #对应-bp
    -tc #对应-bc
    -ti #对应-bi
    -ta #对应-ba
    -tb #对应-bb
    -ts #对应-bs
3.  从源码包建立
    --rebuild  #建立二进制包，通-bb
    --recompile  #同-bi
4.  其他的一些选项
    --buildroot=DIRECTORY   #确定以root目录建立包
    --clean  #完成打包后清除BUILD下的文件目录
    --nobuild  #不进行%build的阶段
    --nodeps  #不检查建立包时的关联文件
    --nodirtokens
    --rmsource  #完成打包后清除SOURCES
    --rmspec #完成打包后清除SPEC
    --short-cricuit
    --target=CPU-VENDOR-OS #确定包的最终使用平台
```

#### spec文档的编写  

|          参数          |                                                                                                                                                                           描述                                                                                                                                                                            |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name                   | 软件包的名称，后面可使用`%{name}`的方式引用，具体命令需跟源包一致                                                                                                                                                                                                                                                                                               |
| Summary                | 软件包的内容概要                                                                                                                                                                                                                                                                                                                                            |
| Version                | 软件的实际版本号，具体命令需跟源包一致                                                                                                                                                                                                                                                                                                                        |
| Release                | 发布序列号，具体命令需跟源包一致                                                                                                                                                                                                                                                                                                                             |
| Group                  | 软件分组，建议使用标准分组                                                                                                                                                                                                                                                                                                                                   |
| License                | 软件授权方式，通常就是GPL                                                                                                                                                                                                                                                                                                                                   |
| Source                 | 源代码包，可以带多个用Source1、Source2等源，后面也可以用`%{source1}、%{source2}`引用                                                                                                                                                                                                                                                                           |
| BuildRoot              | 这个是安装或编译时使用的“虚拟目录”，考虑到多用户的环境，一般定义为：`%{_tmppath}/%{name}-%{version}-%{release}-root` 或`%{_tmppath}/%{name}-%{version}-%{release}-buildroot-%(%{__id_u} -n}` 该参数非常重要，因为在生成rpm的过程中，执行`make install`时就会把软件安装到上述的路径中，在打包的时候，同样依赖“虚拟目录”为“根目录”进行操作。后面可使用`$RPM_BUILD_ROOT` 方式引用。 |
| URL                    | 软件的主页                                                                                                                                                                                                                                                                                                                                                 |
| Vendor                 | 发行商或打包组织的信息，例如RedFlag Co,Ltd                                                                                                                                                                                                                                                                                                                   |
| Disstribution          | 发行版标识                                                                                                                                                                                                                                                                                                                                                 |
| Patch                  | 补丁源码，可使用Patch1、Patch2等标识多个补丁，使用`%patch0`或`%{patch0}`引用                                                                                                                                                                                                                                                                                   |
| Prefix: %{_prefix}     | 这个主要是为了解决今后安装rpm包时，并不一定把软件安装到rpm中打包的目录的情况。这样，必须在这里定义该标识，并在编写`%install`脚本的时候引用，才能实现rpm安装时重新指定位置的功能                                                                                                                                                                                            |
| Prefix: %{_sysconfdir} | 这个原因和上面的一样，但由于`%{_prefix}`指/usr，而对于其他的文件，例如/etc下的配置文件，则需要用`%{_sysconfdir}`标识                                                                                                                                                                                                                                               |
| Build Arch             | 指编译的目标处理器架构，noarch标识不指定，但通常都是以`/usr/lib/rpm/marcros`中的内容为默认值                                                                                                                                                                                                                                                                     |
| Requires               | 该rpm包所依赖的软件包名称，可以用>=或<=表示大于或小于某一特定版本，例如：`libpng-devel >= 1.0.20 zlib`   `>=`号两边需用空格隔开，而不同软件名称也用空格分开还有例如PreReq、Requires(pre)、Requires(post)、Requires(preun)、Requires(postun)、                                                                                                                           |
| BuildRequires          | 等都是针对不同阶段的依赖指定                                                                                                                                                                                                                                                                                                                                 |
| Provides               | 指明本软件一些特定的功能，以便其他rpm识别                                                                                                                                                                                                                                                                                                                     |
| Packager               | 打包者的信息                                                                                                                                                                                                                                                                                                                                               |


```console
软件包所属类别Group，具体类别有：
Amusements/Games （娱乐/游戏）
Amusements/Graphics（娱乐/图形）
Applications/Archiving （应用/文档）
Applications/Communications（应用/通讯）
Applications/Databases （应用/数据库）
Applications/Editors （应用/编辑器）
Applications/Emulators （应用/仿真器）
Applications/Engineering （应用/工程）
Applications/File （应用/文件）
Applications/Internet （应用/因特网）
Applications/Multimedia（应用/多媒体）
Applications/Productivity （应用/产品）
Applications/Publishing（应用/印刷）
Applications/System（应用/系统）
Applications/Text （应用/文本）
Development/Debuggers （开发/调试器）
Development/Languages （开发/语言）
Development/Libraries （开发/函数库）
Development/System （开发/系统）
Development/Tools （开发/工具）
Documentation （文档）
System Environment/Base（系统环境/基础）
System Environment/Daemons （系统环境/守护）
System Environment/Kernel （系统环境/内核）
System Environment/Libraries （系统环境/函数库）
System Environment/Shells （系统环境/接口）
User Interface/Desktops（用户界面/桌面）
User Interface/X （用户界面/X窗口）
User Interface/X Hardware Support （用户界面/X硬件支持）

```