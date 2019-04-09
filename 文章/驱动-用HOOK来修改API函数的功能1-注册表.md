我们知道编程实际上是使用各种API函数来达到我们想要的目的。换句话说就是API函数是我们通常编程时使用到的最底层函数。很多人也觉得除了API函数微软没有在提供其它的编程接口。其实微软出了提供API函数意外还提供了另外的一套函数，不过这些函数会随着操作系统的不同有细微的改变。由于这些函数是如此的“不稳定”，所以微软并没有将它们文档化。我们称之为“未文档化函数”。我们通常使用的API函数其实都是使用这些函数来实现的。所以如果我们使用HOOK来修改这些函数的某些执行特征，那么我们就可以实现一些特殊的功能，例如注册表的禁止修改，禁止删除等等。  
从今天开始我会连续的写出如何使用HOOK的方法来修改API函数的一些特性。  
首先我来说说如何做到注册表的禁止修改。  
在“未文档化函数”中有这样一个函数ZWSETVALUEKEY它的定义是这样的：  

```c++
NTSYSAPI NTSTATUS ZwSetValueKey(
     IN HANDLE  KeyHandle,
     IN PUNICODE_STRING  ValueName,
     IN ULONG  TitleIndex  OPTIONAL,
     IN ULONG  Type,
     IN PVOID  Data,
     IN ULONG  DataSize
 );
ZWSETVALUEKEY ZwSetValueKey;
```

这个函数就是用来使用对注册表内容的修改的，也就是说如果我们打开注册表以后对任何的一个键值的修改，系统都会调用这个函数来实现。如果我们将这个函数修改成“如果用户想修改制定的键值的时候，我们不予执行”，那么我们就实现了对这个注册表键值的保护。那如何来实现呢？下面是我实现的代码和讲解，希望对大家有所帮助。  

```c++
#include "ntddk.h"
#include "bugcodes.h"
#include "ntstatus.h"
#include <ntddkbd.h>
#include <stdio.h>
#include "stdarg.h"
#include "ntiologc.h"
#define MAXPATHLEN 1024
#define MaxBuf 1024
#define MIN(x,y) ((x) < (y) ? (x) : (y))
.....

//(1).声明原有函数
typedef NTSTATUS (*REALZWSETVALUEKEY)(
 IN HANDLE  KeyHandle,
 IN PUNICODE_STRING  ValueName,
 IN ULONG  TitleIndex  OPTIONAL,
 IN ULONG  Type,
 IN PVOID  Data,
 IN ULONG  DataSize
 );
REALZWSETVALUEKEY RealZwSetValueKey;
//(2).定义HOOK注册表设置内容的函数
NTSTATUS HookZwSetValueKey(
 IN HANDLE  KeyHandle,
 IN PUNICODE_STRING  ValueName,
 IN ULONG  TitleIndex  OPTIONAL,
 IN ULONG  Type,
 IN PVOID  Data,
 IN ULONG  DataSize
 );
...
//驱动的入口函数
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) 
{
......
RealZwSetValueKey=(REALZWSETVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetValueKey")]);
  (REALZWSETVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetValueKey")])=HookZwSetValueKey;
// 上面的2行代码就是我们将函数ZwSetValueKey替换成我们自己定义的函数HookZwSetValueKey，这个时候如果有系统要调用函数ZwSetValueKey的时候就会调用我们定义的函数HookZwSetValueKey。而在这个函数中所做的就是我们对修改键值的判断。
}
PVOID GetPointer( HANDLE handle )
{
  PVOID         pKey;
  if(!handle) return NULL;
  //取得指针.
  if( ObReferenceObjectByHandle( handle, 0, NULL, KernelMode, &pKey, NULL ) != STATUS_SUCCESS ) 
  {
      pKey = NULL;
  } 
  return pKey;
}
//HOOK设置注册表键值的函数
NTSTATUS HookZwSetValueKey(
  IN HANDLE  KeyHandle,
  IN PUNICODE_STRING  ValueName,
  IN ULONG  TitleIndex  OPTIONAL,
  IN ULONG  Type,
  IN PVOID  Data,
  IN ULONG  DataSize)
{
 NTSTATUS rc;
 UNICODE_STRING *pUniName;  //定义得到修改注册表的UNI路径
 ULONG actualLen;
 ANSI_STRING keyname,
    akeyname,
    m_keyname,
    m_akeyname;       //定义得到修改注册表的UNI路径
 PVOID pKey;
 RtlUnicodeStringToAnsiString( &akeyname, ValueName, TRUE);
 RtlUnicodeStringToAnsiString( &m_akeyname, ValueName, TRUE);
 RtlUpperString(&akeyname,&m_akeyname);
 RtlFreeAnsiString(&m_akeyname);
 if( pKey = GetPointer( KeyHandle))
 {
  pUniName = ExAllocatePool( NonPagedPool, 512*2+2*sizeof(ULONG));
  pUniName->MaximumLength = 512*2;
  if( NT_SUCCESS( ObQueryNameString( pKey, pUniName, MAXPATHLEN, &actualLen)))
  {
   RtlUnicodeStringToAnsiString( &keyname, pUniName, TRUE);
   keyname.Buffer=_strupr(keyname.Buffer);
   akeyname.Buffer=_strupr(akeyname.Buffer);
   RtlUnicodeStringToAnsiString( &m_keyname, pUniName, TRUE);
   RtlUpperString(&keyname,&m_keyname);
   RtlFreeAnsiString(&m_keyname);
   if (strcmp(keyname.Buffer,\\REGISTRY\\MACHINE\\SOFTWARE\\TEST) == 0)
   {
    if(strcmp(akeyname.Buffer,"TEST") ==0)
    {
     RtlFreeAnsiString(&akeyname); 
     RtlFreeAnsiString(&keyname); 
     return 0;
    }
   }
   else if (strcmp(keyname.Buffer,"\\REGISTRY\\MACHINE\\SOFTWARE\\MICROSOFT\\WINDOWS\\CURRENTVERSION\\RUN") == 0)
   {
    if(strcmp(akeyname.Buffer,"TEST2") ==0)
    {
     RtlFreeAnsiString(&akeyname); 
     RtlFreeAnsiString(&keyname); 
     return 0;
    }
   }
   RtlFreeAnsiString(&keyname); 
  }
 }
 RtlFreeAnsiString(&akeyname); 
 rc=RealZwSetValueKey(KeyHandle,ValueName,TitleIndex,Type,Data,DataSize);
 return (rc);
}
```
上面的代码就是实现了对注册表中TEST下的Test键值和对RUN下的TEST2键值进行保护。
以上的代码是我自己开发的一个软件中实际使用的代码，是经过测试的！