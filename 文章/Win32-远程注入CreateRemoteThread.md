要实现线程的远程注入必须使用Windows提供的`CreateRemoteThread`函数来创建一个远程线程  
该函数的原型如下：  
```c++
HANDLE CreateRemoteThread(
     HANDLE hProcess,    
     LPSECURITY_ATTRIBUTES lpThreadAttributes,
     SIZE_T dwStackSize,
     LPTHREAD_START_ROUTINE lpStartAddress,
     LPVOID lpParameter,
     DWORD dwCreationFlags,
     LPDWORD lpThreadId
);
参数说明：
    hProcess：目标进程的句柄
    lpThreadAttributes：指向线程的安全描述结构体的指针，一般设置为NULL，表示使用默认的安全级别
    dwStackSize：线程堆栈大小，一般设置为0，表示使用默认的大小，一般为1M
    lpStartAddress：线程函数的地址
    lpParameter：线程参数
    dwCreationFlags：线程的创建方式
    CREATE_SUSPENDED 线程以挂起方式创建
    lpThreadId：输出参数，记录创建的远程线程的ID
```

CreateRemoteThread函数介绍完毕，其他详细信息参考MSDN中关于该函数的详细说明！  
既然知道了使用这个函数来创建一个远程线程，接下来我们就来定义线程函数体，和普通的线程函数的定义相同，远程线程的线程函数必须定义程类的  
静态成员函数或者全局函数,例如：  
```c++
DWORD __stdcall threadProc(LPVOID lParam)
{
    //我们在这里先将该线程函数定义为空函数
    return 0;
}
```

在这里我们先将线程函数体定义为空，因为要作为远程注入的线程，线程函数体的编写方式和普通线程函数稍有不同。  
然后将线程代码拷贝到目标进程地址空间中（该地址必须是页面属性为`PAGE_EXECUTE_READWRITE`的页面）或者其他宿主进程能执行地方（如：共享内存映射区）。
在这里我们选择宿主进程。在拷贝线程体的时候我们需要使用`VirtualAllocEx`函数在宿主进程中申请一块存储区域，然后再通过`WriteProcessMemory`函数将线程代码  
写入宿主进程中。  
要取得宿主进程的ID可以有很多种方法可以使用`Psapi.h`中的函数，也可以使用Toolhelp函数，在这里提供一种使用Toolhelp实现的函数，函数如下  
```c++
//根据进程名称得到进程的ID，如果有多个实例在同时运行的话，只返回第一个枚举到的进程ID
DWORD processNameToId(LPCTSTR lpszProcessName)
{
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(PROCESSENTRY32);
    if (!Process32First(hSnapshot, &pe)) {
        MessageBox(NULL,
            "The frist entry of the process list has not been copyied to the buffer",
            "Notice", MB_ICONINFORMATION | MB_OK);
        return 0;
    }
     while (Process32Next(hSnapshot, &pe)) {
         if (!strcmp(lpszProcessName, pe.szExeFile)) {
             return pe.th32ProcessID;
         }
     }

     return 0;
}
```
以上步骤完成之后就可以使用`CreateRemoteThread`创建远程线程了！示例代码如下  
```c++
#include <windows.h>
#include <TlHelp32.h>
#include <iostream>
//要插入宿主进程中的线程函数
DWORD __stdcall threadProc(LPVOID lParam)
{
     return 0;
}
int main(int argc, char* argv[])
{
     const DWORD dwThreadSize = 4096;
     DWORD dwWriteBytes;
     std::cout << "Please input the name of target process" << std::endl;
     char szExeName[MAX_PATH] = { 0 };
     //等待输入宿主进程名称
     std::cin >> szExeName;

     //得到指定名称进程的进程ID，如果有多个进程实例，则得到第一个进程ID
     DWORD dwProcessId = processNameToId(szExeName);
     HANDLE hTargetProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId)
     void* pRemoteThread = VirtualAllocEx(hTargetProcess, 0,
     dwThreadSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
     //把线程体写入宿主进程中
     if (!WriteProcessMemory(hTargetProcess,
         pRemoteThread, &threadProc, dwThreadSize, 0)) {
         MessageBox(NULL, "Write data to target process failed !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }
     //在宿主进程中创建线程
     HANDLE hRemoteThread = CreateRemoteThread(
         hTargetProcess, NULL, 0, (DWORD (__stdcall *)(void *))pRemoteThread,
         NULL, 0, &dwWriteBytes);
     if (!hRemoteThread) {
         MessageBox(NULL, "Create remote thread failed !", "Notice", MB_ICONSTOP);
         return -1;
     }
     return 0;
}
```
当上面的代码运行的时候会在宿主进程中创建一条由程序员定义的线程，只不过现在这个线程函数体为空什么都不做。  
下面我们来编写具体的线程函数体的内容，在这里我们只是简单的显示一个消息对话框`MessageBox`修改之后的线程函数体如下：  
```c++
DWORD __stdcall threadProc(LPVOID lParam)
{
     MessageBox(NULL, "hello", "hello", MB_OK);
     return 0;
}
```
线程体修改完毕之后我们运行程序，将线程注入到宿主进程之中。不过此时会产生一个非法访问的错误。原因就是线程体中的`MessageBox(NULL, "hello", "hello", MB_OK);`函数的第二和第三个参数所指向的字符串是存在于当前进程的地址空间中，`宿主进程`中的`线程`访问该字符串"hello"就会出现`访问内存非法`的错误。解决的  
方法就是`将该字符串的内容也拷贝到宿主进程的地址空间中`，而且连同MessageBox函数在User32.dll中的地址也拷贝到宿主进程之中。  
要将字符串和MessageBox函数的入口地址拷贝到宿主进程中我们首先定义下面这个RemoteParam结构体，用来存放MessageBox函数的入口地址和MessageBox显示的字符串的内容，该结构的定义如下：  
```c++
//线程参数
typedef struct _RemoteParam {
     char szMsg[12];     //MessageBox函数显示的字符串
     DWORD dwMessageBox;//MessageBox函数的入口地址
} RemoteParam, * PRemoteParam;
RemoteParam remoteData;
ZeroMemory(&remoteData, sizeof(RemoteParam));

HINSTANCE hUser32         = LoadLibrary("User32.dll");
remoteData.dwMessageBox   = (DWORD)GetProcAddress(hUser32, "MessageBoxA");
strcat(remoteData.szMsg, "Hello\0");
//在宿主进程中分配存储空间
RemoteParam* pRemoteParam = (RemoteParam*)VirtualAllocEx(hTargetProcess , 0, sizeof(RemoteParam), MEM_COMMIT, PAGE_READWRITE);
if (!pRemoteParam) {
     MessageBox(NULL, "Alloc memory failed !", "Notice", MB_ICONINFORMATION | MB_OK);
     return 0;
}
//将字符串和MessageBox函数的入口地址写入宿主进程
if (!WriteProcessMemory(hTargetProcess ,pRemoteParam, &remoteData, sizeof(remoteData), 0)) {
     MessageBox(NULL, "Write data to target process failed !","Notice", MB_ICONINFORMATION | MB_OK);
     return 0;
}

//创建远程线程
HANDLE hRemoteThread = CreateRemoteThread(hTargetProcess, NULL, 0, (DWORD (__stdcall *)(void *))pRemoteThread,
     pRemoteParam, 0, &dwWriteBytes);
```
另外还需要注意的一点是，在打开进程的时候`有些系统进程`是无法用`OpenProcess函数打开`的，这个时候就需要提升进程的访问权限，进而来达到访问系统进程的目的，  
在这里我提供了一个提升进程访问权限的函数`enableDebugPriv()`,该函数的定义如下：  
```c++
//提升进程访问权限
bool enableDebugPriv()
{
     HANDLE hToken;
     LUID sedebugnameValue;
     TOKEN_PRIVILEGES tkp;
  
     if (!OpenProcessToken(GetCurrentProcess(),
         TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken)) {
         return false;
     }
     if (!LookupPrivilegeValue(NULL, SE_DEBUG_NAME, &sedebugnameValue)) {
         CloseHandle(hToken);
         return false;
     }
     tkp.PrivilegeCount = 1;
     tkp.Privileges[0].Luid = sedebugnameValue;
     tkp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
     if (!AdjustTokenPrivileges(hToken, FALSE, &tkp, sizeof(tkp), NULL, NULL)) {
         CloseHandle(hToken);
         return false;
     }
     return true;
}
```

至此创建远程线程的工作全部结束，下面就给出完整的代码：  
```c++
#pragma once
#include "stdafx.h"
#include <windows.h>
#include <TlHelp32.h>
#include <iostream>
//线程参数结构体定义
typedef struct _RemoteParam {
     char szMsg[12];     //MessageBox函数中显示的字符提示
     DWORD dwMessageBox;//MessageBox函数的入口地址
} RemoteParam, * PRemoteParam;
//定义MessageBox类型的函数指针
typedef int (__stdcall * PFN_MESSAGEBOX)(HWND, LPCTSTR, LPCTSTR, DWORD);

//线程函数定义
DWORD __stdcall threadProc(LPVOID lParam)
{
     RemoteParam* pRP = (RemoteParam*)lParam;

     PFN_MESSAGEBOX pfnMessageBox;
     pfnMessageBox = (PFN_MESSAGEBOX)pRP->dwMessageBox;
     pfnMessageBox(NULL, pRP->szMsg, pRP->szMsg, 0);
     return 0;
}
//提升进程访问权限
bool enableDebugPriv()
{
     HANDLE hToken;
     LUID sedebugnameValue;
     TOKEN_PRIVILEGES tkp;
  
     if (!OpenProcessToken(GetCurrentProcess(),
         TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken)) {
         return false;
     }
     if (!LookupPrivilegeValue(NULL, SE_DEBUG_NAME, &sedebugnameValue)) {
         CloseHandle(hToken);
         return false;
     }
     tkp.PrivilegeCount = 1;
     tkp.Privileges[0].Luid = sedebugnameValue;
     tkp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
     if (!AdjustTokenPrivileges(hToken, FALSE, &tkp, sizeof(tkp), NULL, NULL)) {
         CloseHandle(hToken);
         return false;
     }
     return true;
}

//根据进程名称得到进程ID,如果有多个运行实例的话，返回第一个枚举到的进程的ID
DWORD processNameToId(LPCTSTR lpszProcessName)
{
     HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
     PROCESSENTRY32 pe;
     pe.dwSize = sizeof(PROCESSENTRY32);
     if (!Process32First(hSnapshot, &pe)) {
         MessageBox(NULL,
             "The frist entry of the process list has not been copyied to the buffer",
            "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }
     while (Process32Next(hSnapshot, &pe)) {
         if (!strcmp(lpszProcessName, pe.szExeFile)) {
             return pe.th32ProcessID;
         }
     }

     return 0;
}
int main(int argc, char* argv[])
{
     //定义线程体的大小
     const DWORD dwThreadSize = 4096;
     DWORD dwWriteBytes;
     //提升进程访问权限
     enableDebugPriv();
     //等待输入进程名称，注意大小写匹配
     std::cout << "Please input the name of target process !" << std::endl;
     char szExeName[MAX_PATH] = { 0 };
     std::cin >> szExeName;
     DWORD dwProcessId = processNameToId(szExeName);
     if (dwProcessId == 0) {
         MessageBox(NULL, "The target process have not been found !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return -1;
     }
     //根据进程ID得到进程句柄
     HANDLE hTargetProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId);

     if (!hTargetProcess) {
         MessageBox(NULL, "Open target process failed !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }

     //在宿主进程中为线程体开辟一块存储区域
     //在这里需要注意MEM_COMMIT | MEM_RESERVE内存非配类型以及PAGE_EXECUTE_READWRITE内存保护类型
     //其具体含义请参考MSDN中关于VirtualAllocEx函数的说明。
     void* pRemoteThread = VirtualAllocEx(hTargetProcess, 0,
         dwThreadSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
     if (!pRemoteThread) {
         MessageBox(NULL, "Alloc memory in target process failed !",
             "notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }

     //将线程体拷贝到宿主进程中
     if (!WriteProcessMemory(hTargetProcess,
             pRemoteThread, &threadProc, dwThreadSize, 0)) {
         MessageBox(NULL, "Write data to target process failed !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }
     //定义线程参数结构体变量
     RemoteParam remoteData;
     ZeroMemory(&remoteData, sizeof(RemoteParam));

     //填充结构体变量中的成员
     HINSTANCE hUser32 = LoadLibrary("User32.dll");
     remoteData.dwMessageBox = (DWORD)GetProcAddress(hUser32, "MessageBoxA");
     strcat(remoteData.szMsg, "Hello\0");

     //为线程参数在宿主进程中开辟存储区域
     RemoteParam* pRemoteParam = (RemoteParam*)VirtualAllocEx(
     hTargetProcess , 0, sizeof(RemoteParam), MEM_COMMIT, PAGE_READWRITE);

     if (!pRemoteParam) {
         MessageBox(NULL, "Alloc memory failed !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }
     //将线程参数拷贝到宿主进程地址空间中
     if (!WriteProcessMemory(hTargetProcess ,
             pRemoteParam, &remoteData, sizeof(remoteData), 0)) {
         MessageBox(NULL, "Write data to target process failed !",
             "Notice", MB_ICONINFORMATION | MB_OK);
         return 0;
     }

     //在宿主进程中创建线程
     HANDLE hRemoteThread = CreateRemoteThread(
         hTargetProcess, NULL, 0, (DWORD (__stdcall *)(void *))pRemoteThread,
         pRemoteParam, 0, &dwWriteBytes);
     if (!hRemoteThread) {
         MessageBox(NULL, "Create remote thread failed !", "Notice",   MB_ICONINFORMATION | MB_OK);
         return 0;
     }
     CloseHandle(hRemoteThread);
     return 0;
}
```