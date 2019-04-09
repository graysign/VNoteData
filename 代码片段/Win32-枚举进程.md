
在Windows 2000以上的MS操作系统，通过Windows的任务管理器可以列出当前系统的所有活动进程,在Windows XP中，更是在控制台下增加了一条Tasklist命令，让系统下的所有进程无所遁行。这一切是怎么实现的呢？
方法一

第一种方法是大家比较熟悉的通过ToolHelp Service提供的API函数来实现。这里用到了3个关键的函数：CreateToolhelp32Snapshot()，Process32First()和Process32Next()。下面给出了关于这三个函数的原形和参数说明;
```c++
HANDLE WINAPI CreateToolhelp32Snapshot(
   DWORD dwFlags, //系统快照要查看的信息类型
   DWORD th32ProcessID       //值0表示当前进程
);
BOOL WINAPI Process32First(
   HANDLE hSnapshot,         //CreateToolhelp32Snapshot()创建的快照句柄
   LPPROCESSENTRY32 lppe   //指向进程入口结构
);
BOOL WINAPI Process32Next(
   HANDLE hSnapshot,         //这里参数同Process32First
   LPPROCESSENTRY32 lppe   //同上
);
```
首先使用CreateToolhelp32Snapshot()创建系统快照句柄（hprocess是我们声明用来保存创建的快照句柄）：
```c++
hProcess=CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,0);
```
然后调用Process32First()获得系统快照中的第一个进程信息（Report是BOOL型作为判断系统快照中下一条进程记录）：

```c++
report=Process32First(hProcess,pinfo);
```
接着用一个循环调用来遍历系统中所有运行的进程：
```c++
while(report)
{
hModule=CreateToolhelp32Snapshot(TH32CS_SNAPMODULE,pinfo->th32ProcessID);
   Module32First(hModule, minfo); 
   GetShortPathName(minfo->szExePath,shortpath,256);
   printf("%s --- %s/n",pinfo->szExeFile,shortpath);
   report=Process32Next(hProcess, pinfo);   
}
```
笔者曾通过对Pstools工具包里的Pslist.exe反编译，发现该工具用的就是这种方法。如果你查询MSDN，可以找到一个比这个功能更加完善的源程序。

方法二

第二种方法也很常见，通过MSDN就可以找到例子代码，它是通过Psapi.dll提供的API函数EnumProcesses和EnumProcessModules来实现。有一点要说明的是，Visual Studio提供的SDK包里没有提供相应的Psapi.h和与之相对应的导入库，笔者当时就很纳闷，MSDN里的例子居然编译不通，后来才发现，编译时根本找不到“psapi.h”。呵呵，还好，MSDN至少告诉我们他们都包含Psapi.dll里，用VC自带的Depend工具一查，果然。这样就好办了，我们可以自己找到这些函数入口地址。
小知识：C也好C++也好，一般的函数名本质上都是一个地址，在Win32的API里，它是指向函数所在Dll模块里函数实现的入口地址。
下面是第二种方法的实现过程：首先，先把Psapi.dll里要用到函数都定义好，方便后面显示调用。

//在psaipi.dll中的函数EnumProcesses用来枚举进程 
```c++
typedef BOOL (_stdcall *ENUMPROCESSES)(   //注意这里要指明调用约定为-stdcall
DWORD* pProcessIds,   //指向进程ID数组链  
DWORD cb,     //ID数组的大小，用字节计数
DWORD* pBytesReturned);    //返回的字节
//在psapi.dll中的函数EnumProcessModules用来枚举进程模块
typedef BOOL (_stdcall *ENUMPROCESSMODULES)(
HANDLE hProcess,    //进程句柄
HMODULE* lphModule, //指向模块句柄数组链
DWORD cb,     //模块句柄数组大小，字节计数
LPDWORD lpcbNeeded);    //存储所有模块句柄所需的字节数
//在psapi.dll中的函数GetModuleFileNameEx获得进程模块名
typedef DWORD (_stdcall *GETMODULEFILENAMEEX)(
HANDLE hProcess,    //进程句柄
HMODULE hModule,    //进程句柄
LPTSTR lpFilename,    //存放模块全路径名
DWORD nSize     //lpFilename缓冲区大小，字符计算
);
```
好，通过每个定义的函数前的注释你可以清楚看到这些原函数名，然后当然要加载包含这些函数模块Psapi.dll啦：
```c++
hPsDll = LoadLibrary("PSAPI.DLL");
```
接着，通过Dll入口，我们查找每个函数地址：
```c++
pEnumProcesses           =     (ENUMPROCESSES)GetProcAddress(hPsDll, "EnumProcesses");
pEnumProcessModules      =     (ENUMPROCESSMODULES)GetProcAddress(hPsDll, "EnumProcessModules");
pGetModuleFileNameEx     =     (GETMODULEFILENAMEEX)GetProcAddress(hPsDll, "GetModuleFileNameExA");
```
注意第三个函数名GetModuleFileNameExA，在Dll里有以A和W结尾区分函数，A指采用的是ANSI字符串方式，W则是UNICODE方式。于是，我们可以用下面的语句枚举进程：
```c++
pEnumProcesses(processid, sizeof(processid), &needed);
processcount    =    needed/sizeof(DWORD);
for (i=0;i<processcount;i++){//打开进程
   hProcess=OpenProcess(PROCESS_QUERY_INFORMATION|PROCESS_VM_READ,false, processid[i]);
   if (hProcess)
   {
        pEnumProcessModules(hProcess, &hModule, sizeof(hModule), &needed);
        pGetModuleFileNameEx(hProcess, hModule, path, sizeof(path));
        GetShortPathName(path,path,256);
        itoa(processid[i],temp,10);
        printf("%s --- %s/n",path,temp);
   }else{
    printf("Failed!!!/n");
   }
}
```
当然，在Google上一搜索，可以找到不少提供的Psapi.h头文件和对应的Psapi.lib库，这样，你就可以更方便用这种方法了。
方法三

也许你会说前两种方法全世界都知道了，没什么大不了的！呵呵，那么笔者现在介绍的第三种方法，你就未必知道了。本方法利用了Windows NT/2000下终端服务API函数WTSOpenServer()和WTSEnumerateProcess()来实现，这两个函数都定义在Wtsapi32.dll里。具体的关于终端服务方面的知识，大家可以查询MSDN。
首先，我们来显示申明这两个函数原形：
```c++
typedef HANDLE (_stdcall *WTSOPENSERVER)(
LPTSTR pServerName //NetBios指定的终端服务名，如果我们查看本地终端所有进程信息我们可以通过在控制台命令行下用nbtstat –an来获取本机NetBios名。如图3所示。
);  
```
四种方法实现VC枚举系统当前进程

图3
```c++
typedef BOOL (_stdcall *WTSENUMERATEPROCESSES)(
   HANDLE hServer, //WTSOpenServer返回的句柄 
   DWORD Reserved,   //保留值， 0
   DWORD Version,    //指定枚举要求的版本， 必须为1
   PWTS_PROCESS_INFO* ppProcessInfo, //这个参数是关键，存放我们要的进程名和进程id
   DWORD* pCount    //用来存放ppProcessInfo里WTS_PROCESS_INFO结构的数量指针
);
```
和前面一样，要先装载Wtsapi32.dll模块，获取关键函数地址：
```c++
hWtsApi32                  =     LoadLibrary("wtsapi32.dll");
pWtsOpenServer             =     (WTSOPENSERVER)GetProcAddress(hWtsApi32, "WTSOpenServerA");
pWtsEnumerateProcesses     =     (WTSENUMERATEPROCESSES)GetProcAddress(hWtsApi32,"WTSEnumerateProcessesA");

//通过Argv[1]给终端服务名（这里我们赋本机NetBios名）赋一个值并打开这项服务：
char* szServerName         =      argv[1];
hWtsServer                 =      pWtsOpenServer(szServerName);
//然后开始遍历终端服务器上的所有进程，这里我们是指本机的所有进程
if(!pWtsEnumerateProcesses(hWtsServer, 0, 1, &pWtspi, &dwCount))
{
   printf("enum processes error: %d/n", GetLastError());
   return;
};
for(int i=0; i<dwCount; i++)
{
   printf("ps_Id: %d/t/tps_name: %s/n", pWtspi[i].ProcessId, pWtspi[i].pProcessName);
}
```
怎么样，酷吧，跟前面两种方法效果一样！

方法四

最后笔者介绍一种最少人用的方法，本方法是从“幻影旅团”论坛上看到的。后面给出的源代码也是从他们那拷贝过来的。呵呵，这种方法也提供给我们一种很好的思路。

这第四种方法利用了Native Api的NtQuerySystemInformation函数来实现。同样没有该函数的导入库，也要自己定义原形。整个实现不难，可是有点烦，因为在该函数参数结构上，笔者通查MSDN查了很久才找到这些相关的结构，下面我们来看看这个方法的实现吧。
先是自定义函数原形：
```c++
typedef NTSTATUS (__stdcall *PZWQUERYSYSTEMINFORMATION) 
                  (IN SYSTEM_INFORMATION_CLASS SystemInformationClass,  
                   IN OUT PVOID SystemInformation,  
                   IN ULONG SystemInformationLength,  
                   OUT PULONG ReturnLength);
```                
然后，还是和前面一样，要找到ZwQuerySystemInformation在Ntdll.dll模块里的入口地址：
```c++
hModule                         = GetModuleHandle((LPCTSTR)"ntdll.dll");
pZwQuerySystemInformation       = (PZWQUERYSYSTEMINFORMATION)
GetProcAddress(hModule, (LPCTSTR)"ZwQuerySystemInformation");
获得指向进程信息数组链的第一条进程信息：
pSystemProcessInformation       = (SYSTEM_PROCESS_INFORMATION *)pvProcessList;
和方法一方法二一样，在获得第一条进程信息后，开始循环遍历，列出其余的进程：
while (TRUE){ 
      if (pSystemProcessInformation->NextEntryDelta == 0) //如果是最后一条，则终止循环。
             break;                                    
      pSystemProcessInformation=(SYSTEM_PROCESS_INFORMATION*)((PCHAR)pSystemProcessInformation+ pSystemProcessInformation->NextEntryDelta); 
      pProcessName = (char *)malloc(pSystemProcessInformation->ProcessName.Length + 2); 
}
//特别注意的是，这里SYSTEM_PROCESS_INFORMATION结构里进程名的类型为UNICODE_STRING, 而UNICODE_STRING里的成员Buffer定义的是Unicode类型。
typedef struct _SYSTEM_PROCESS_INFORMATION  
{  
     DWORD NextEntryDelta;  
     DWORD dThreadCount;  
     DWORD dReserved01;  
     DWORD dReserved02;  
     DWORD dReserved03;  
     DWORD dReserved04;  
     DWORD dReserved05;  
     DWORD dReserved06;  
     FILETIME ftCreateTime; /* relative to 01-01-1601 */  
     FILETIME ftUserTime; /* 100 nsec units */  
     FILETIME ftKernelTime; /* 100 nsec units */  
     UNICODE_STRING ProcessName;       //这就是进程名
     DWORD BasePriority;  
     DWORD dUniqueProcessId;             //进程ID
     DWORD dParentProcessID;  
     DWORD dHandleCount;  
     DWORD dReserved07;  
     DWORD dReserved08;  
     DWORD VmCounters;  
     DWORD dCommitCharge;  
     PVOID ThreadInfos[1]; 
} SYSTEM_PROCESS_INFORMATION, *PSYSTEM_PROCESS_INFORMATION;
typedef struct _UNICODE_STRING { 
   USHORT Length; 
   USHORT MaximumLength; 
   PWSTR   Buffer;                  //注意，这里为Unicode类型
} UNICODE_STRING, *PUNICODE_STRING;
//所以，最后我们要调用WideCharToMultiByte（）来还原成ANSI字符串以便输出：
WideCharToMultiByte(CP_ACP,  
                       0, 
                     pSystemProcessInformation->ProcessName.Buffer, 
                     pSystemProcessInformation->ProcessName.Length + 1, 
                     pProcessName, 
                     pSystemProcessInformation->ProcessName.Length + 1, 
                     NULL, 
                     NULL);
```
具体每个参数大家可以参照MSDN，很简单的。以上四种方法的完整源代码见光盘。
从这里看出，查看系统当前活动进程的实现也不过如此，况且，笔者还没有给出第五种方法（可以通过WMI的COM接口函数CoCreateInstance()，ConnectServer()，ExecQuery()等函数来实现，留给读者自己练习），呵呵，好玩吧？居然有那么多种方法。

写这篇文章的目的是想告诉大家，有时候通过自己动手，会发现以前认为很NB的东西原来也是很简单的，只要擅于总结，常动手，我们自己也可以自己打造一些方便好用的小工具来，你说呢？！