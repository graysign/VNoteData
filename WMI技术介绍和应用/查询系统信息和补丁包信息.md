&emsp;&emsp; 本文主要知识点是[Win32_OperatingSystem](http://msdn.microsoft.com/en-us/library/windows/desktop/aa394239(v=vs.85).aspx)和[Win32_QuickFixEngineering](http://msdn.microsoft.com/en-us/library/windows/desktop/aa394391(v=vs.85).aspx)类。通过该类我们将可以获取部分系统设置。  

&emsp;&emsp;**如何使用WMI获取系统UUID？**  
```
SELECT UUID FROM Win32_ComputerSystemProduct  
```
![](_v_images/_1521445608_18350.png)  
&emsp;&emsp;该值可能为空。如果该值存在，其保存在`HKEY_LOCAL_MACHINE\SOFTWARE\Intel\PIcon\AMTData\System UUID`下。  

&emsp;&emsp;**如何使用WMI获取Windows操作系统启动硬盘设备名？**  
```
SELECT BootDevice FROM Win32_OperatingSystem 
```
![](_v_images/_1521445658_7516.png)  

&emsp;&emsp;**如何使用WMI获取系统Build版本号？**  
```
SELECT BuildNumber FROM Win32_OperatingSystem  
```
![](_v_images/_1521445710_1710.png)  

&emsp;&emsp;**如何使用WMI获取系统Build版本类型？**
```
SELECT BuildType FROM Win32_OperatingSystem 
```
![](_v_images/_1521445761_16920.png)  
&emsp;&emsp;基于NT版本的操作系统又两种Build版本类型。一种是Checked，一种是Free（或者retail）。一般来说，Free版本就是零售版本，其二进制执行文件是经过了编译器优化的。而Checked版本，则是为了方便调试，将很多编译器优化禁用了，还增加了很多的调试检测代码。所以一般来说，Free版本的文件大小要比Checked版本文件大小要小。更详细的资料请参略http://msdn.microsoft.com/en-us/library/ff543450.aspx。  

&emsp;&emsp;**如何使用WMI获取系统名？**  
```
SELECT Caption FROM Win32_OperatingSystem  
```
![](_v_images/_1521445800_18026.png)  

&emsp;&emsp;**如何使用WMI获取系统的Code Page?**  
```
SELECT CodeSet FROM Win32_OperatingSystem 
```
![](_v_images/_1521445837_32687.png)  
&emsp;&emsp;936即对应于Simplified Chinese GBK。  


&emsp;&emsp;**如何使用WMI获取地区代码？**  
```
SELECT CountryCode FROM Win32_OperatingSystem  
```
![](_v_images/_1521445875_863.png)  
&emsp;&emsp;86即对应于中国大陆。台湾地区是886，香港是852，澳门是853。  


&emsp;&emsp;**如何使用WMI获取系统的补丁包版本号？**  
```
SELECT CSDVersion FROM Win32_OperatingSystem  
```
![](_v_images/_1521445908_9025.png)  

&emsp;&emsp;**如何使用WMI获取系统的空闲的物理内存？**  
```
SELECT FreePhysicalMemory FROM Win32_OperatingSystem  
```
![](_v_images/_1521445941_22136.png)  
&emsp;&emsp;该单位是以Kb为单位的。它标识了当前系统有多少尚未使用且可用的内存。  
  
&emsp;&emsp;**如何使用WMI获取页文件空闲空间大小？**  
```
SELECT FreeSpaceInPagingFiles FROM Win32_OperatingSystem  
  
```
![](_v_images/_1521445993_2349.png)  
&emsp;&emsp;该数值也是以Kb为单位的。  

&emsp;&emsp;**如何使用WMI获取空闲的虚拟内存大小？**  
```
SELECT FreeVirtualMemory FROM Win32_OperatingSystem 
```
![](_v_images/_1521446038_15664.png)  
&emsp;&emsp;该数值也是以Kb为单位的。  


&emsp;&emsp;**如何使用WMI获取系统最后一次启动时间？**  
```
SELECT LastBootUpTime FROM Win32_OperatingSystem 
```
![](_v_images/_1521446074_26934.png)  
 &emsp;&emsp;这表示我最近一次系统启动时间是2013年2月4号9时6分22秒。  

&emsp;&emsp;**如何使用WMI获取系统中正在运行的进程数量？**  
```
SELECT NumberOfProcesses FROM Win32_OperatingSystem  
```
![](_v_images/_1521446123_18343.png)  

&emsp;&emsp;**如何使用WMI获取系统注册用户的公司名？**  
```
SELECT Organization FROM Win32_OperatingSystem  
```
![](_v_images/_1521446154_5305.png)  


&emsp;&emsp;**如何使用WMI获取系统语言包种类？**  
```
SELECT OSLanguage FROM Win32_OperatingSystem  
```
![](_v_images/_1521446182_1152.png)  
&emsp;&emsp; 其对应的是Chinese (Simplified) – PRC  


&emsp;&emsp;**如何使用WMI判断系统是否从外置USB设备启动的？**  
```
SELECT PortableOperatingSystem FROM Win32_OperatingSystem  
```
&emsp;&emsp;为True则代表是从USB设备中启动的。  



&emsp;&emsp;**如何使用WMI判断当前系统是否是主系统？**  
```
SELECT Primary FROM Win32_OperatingSystem  
```
![](_v_images/_1521446254_16292.png)  



&emsp;&emsp;**如何使用WMI判断系统类型？**  
```
SELECT ProductType FROM Win32_OperatingSystem  
```
![](_v_images/_1521446284_127.png)  
| Value | Meaning |
| --- | --- |  
| 1 | Work Station |  
| 2 | Domain Controller |  
| 3 | Server |  


&emsp;&emsp;**如何使用WMI获取系统的注册用户名？**  
```
SELECT RegisteredUser FROM Win32_OperatingSystem  
```
![](_v_images/_1521446361_29195.png)  


&emsp;&emsp;**如何使用WMI获取系统序列号？**  
```
SELECT SerialNumber FROM Win32_OperatingSystem  
```
![](_v_images/_1521446394_8254.png)  



&emsp;&emsp;**如何使用WMI获取系统安装在那个设别上？**  
```
SELECT SystemDevice FROM Win32_OperatingSystem  
```
![](_v_images/_1521446429_28563.png)  

&emsp;&emsp;**如何使用WMI获取系统盘盘符？**  
```
SELECT SystemDrive FROM Win32_OperatingSystem  
```
![](_v_images/_1521446465_3191.png)  


&emsp;&emsp;**如何使用WMI查询系统可以见内存大小？**  
```
SELECT TotalVisibleMemorySize FROM Win32_OperatingSystem  
```
![](_v_images/_1521446487_2829.png)  
&emsp;&emsp;  该单位是以Kb为单位的。


&emsp;&emsp;**如何使用WMI枚举已经安装的补丁信息？**  
```
SELECT * FROM Win32_QuickFixEngineering  
```
![](_v_images/_1521446525_28086.png)  
&emsp;&emsp;以上信息是来源于  
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Hotfix  
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Updates  
```