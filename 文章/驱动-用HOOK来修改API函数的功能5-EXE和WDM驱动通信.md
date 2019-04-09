今天我来写关于HOOK修改API函数的最后一篇文章"EXE和WDM驱动通信"。
在上面的几篇文章中，大家看到了被保护的文件名或者注册表键值名等等都是事先指定好的。
但是在实际应用中，可能遇到的是被保护的文件名、注册表的键值名需要动态来指定的情况。
这个时候就需要编写一个上层EXE程序来和WDM驱动通信。
通常EXE和WDM驱动通信有2种方法：
1. 使用DeviceIOControl函数。
2. 使用自定义事件。  

在这里我简述一下使用DeviceIOControl函数的实现方法。  
首先在WDM驱动的.C文件中加入以下的宏定义代码：  
`#define IOCTL_EVENT_MSG   CTL_CODE(FILE_DEVICE_UNKNOWN, 0x966, METHOD_BUFFERED , FILE_ANY_ACCESS)`  
在DriverEntry例程中写入：  
`DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = MydrvDispatchIoctl;`  
例程MydrvDispatchIoctl的实现如下  

```c++
static NTSTATUS MydrvDispatchIoctl(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{ 
 PIO_STACK_LOCATION IrpStack; 
 NTSTATUS status,status1; 
 ULONG ControlCode; 
 ULONG InputLength,OutputLength; 
 TCHAR wInputBuffer[256]; 
 TCHAR OutMsg[] = "Mess";
 UNICODE_STRING *TempInstallDir=NULL;
 PVOID pvIOBuffer;
 char *Test;
 OutputLength=sizeof(OutMsg);
 // 得到当前IRP (IO请求包) 
 IrpStack = IoGetCurrentIrpStackLocation(Irp); 
 // 得到DeviceIoControl传来的功能调用号 
 ControlCode = IrpStack->Parameters.DeviceIoControl.IoControlCode; 
 // 得到DeviceIoControl传来的输入缓冲区长度 
 InputLength = IrpStack->Parameters.DeviceIoControl.InputBufferLength; 
 // 得到DeviceIoControl的输出缓冲区长度 
 OutputLength = IrpStack->Parameters.DeviceIoControl.OutputBufferLength; 
 Test=(char *)Irp->AssociatedIrp.SystemBuffer;
 //使用函数DbgPrint函数将外界EXE函数传入的信息答应出来
 DbgPrint("Test=%s\n",Test);
 switch (ControlCode)
 {   
  case IOCTL_EVENT_MSG:
   //得到应用程序给的参数信息。
   RtlCopyMemory(Irp->AssociatedIrp.SystemBuffer, OutMsg, sizeof(OutMsg)); 
   Irp->IoStatus.Status = STATUS_SUCCESS;
   //设置返回的信息长度
   OutputLength = sizeof(OutMsg);
   //设置返回的信息。
   Irp->IoStatus.Information = OutputLength; 
   break;
 } 
 status = Irp->IoStatus.Status; 
 IoCompleteRequest(Irp, 0); 
 return status; 
} 
```

通过以上的处理，WDM驱动就具有了接收EXE传入信息的能力。  
下面我们再来看看上层EXE部分是如何编写的：  
在我的项目中，我使用VC把与WDM通信的部分写成了一个DLL。  

```c++
#include "stdafx.h"
#include "VCDll.h"
#include <Winsvc.h>
#include "winioctl.h"
#ifdef _DEBUG
#define new DEBUG_NEW
#undef THIS_FILE
static char THIS_FILE[] = __FILE__;
#endif
//注意这里必须和WDM的宏定义部分相同
#define IOCTL_EVENT_MSG   CTL_CODE(FILE_DEVICE_UNKNOWN, 0x966, METHOD_BUFFERED , FILE_ANY_ACCESS)
//参数char * SendInfo是需要传入WDM驱动的信息
int SendToDriver(char * SendInfo)
{
 hcomm=CreateFile("\\\\.\\MyDriver",GENERIC_WRITE,0,NULL,OPEN_EXISTING,FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED,NULL);
 if(hcomm!=INVALID_HANDLE_VALUE)
 {
  char m_dwOptions[255]={'\0'};
  DWORD bytesWrite;
  memset(m_dwOptions,0,sizeof(m_dwOptions));
  memcpy(m_dwOptions,SendInfo,strlen(SendInfo));
  if(DeviceIoControl(hcomm ,IOCTL_EVENT_MSG,( char * )m_dwOptions,strlen(SendInfo)+1,( char * )m_dwOptions, 5, &bytesWrite, NULL ))
  {
   CloseHandle(hcomm);
   return 1;
  }
  else
  {
   CloseHandle(hcomm);
   return 0;
  }
 }
 else
 {
  CloseHandle(hcomm);
  return -1;
 }
}
```
通过以上的处理，WDM就和上层的EXE具有了通信的能力。  
以上的5篇文章就是我在使用DDK来编写保护文件、注册表等等功能项目时使用到的方法。  
我在这里全写出来就是想让更多对这方面感兴趣的朋友参与进来，和我一起交流，大家共同进步。  
以上代码通过了测试并已经长期应用到实践中。