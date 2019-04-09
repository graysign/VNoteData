上次写了如何使用HOOK的方法修改API函数的功能，来对注册表进行保护。对于对注册表操作的函数还有`ZwDeleteKey、ZwDeleteValueKey、ZwOpenKey`等等，对这些函数的HOOK和我上面写的方法是一样的。  
今天我来写一下如何对文件操作的API函数来HOOK。  
    在我们编程中经常使用`CreateFile`函数来创建文件。其实对于系统来说，当我们使用右键点击并选择“新建”中的创建文件的时候，系统也是调用了CreateFile来创建一个文件。对于某些项目来说，我们不想让没有权限的人来随意的来创建文件。这个时候我们就应该对CreateFile函数的功能进行修改。  
        在“未文档化函数”中有一个函数叫做`ZwCreateFile`。这个函数的作用就是创建文件，API函数CreateFile的实现就是调用了这个函数。如果我们修改了这个函数的实现过程，就可以达到修改API函数CreateFile的目的。  
        下面的代码是我给一个大学做的项目中一段。目的是不让学生随便在指定的盘符上创建文件。  

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
typedef NTSTATUS (*REALZWCREATEFILE)(
  OUT PHANDLE FileHandle,
  IN ACCESS_MASK DesiredAccess,
  IN POBJECT_ATTRIBUTES ObjectAttributes,
  OUT PIO_STATUS_BLOCK IoStatusBlock,
  IN PLARGE_INTEGER AllocationSize  OPTIONAL,
  IN ULONG FileAttributes,
  IN ULONG ShareAccess,
  IN ULONG CreateDisposition,
  IN ULONG CreateOptions,
  IN PVOID EaBuffer  OPTIONAL,
  IN ULONG EaLength
  );
REALZWCREATEFILE RealZwCreateFile;
 
NTSTATUS HookZwCreateFile(
  OUT PHANDLE FileHandle,
  IN ACCESS_MASK DesiredAccess,
  IN POBJECT_ATTRIBUTES ObjectAttributes,
  OUT PIO_STATUS_BLOCK IoStatusBlock,
  IN PLARGE_INTEGER AllocationSize  OPTIONAL,
  IN ULONG FileAttributes,
  IN ULONG ShareAccess,
  IN ULONG CreateDisposition,
  IN ULONG CreateOptions,
  IN PVOID EaBuffer  OPTIONAL,
  IN ULONG EaLength
  );
...
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) 
{
...... 
RealZwCreateFile=(REALZWCREATEFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwCreateFile")]);
  (REALZWCREATEFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwCreateFile")])=HookZwCreateFile;
//上面两行的代码含义请参照我上面的文章。
}
 
.....
 
NTSTATUS HookZwCreateFile(
  OUT PHANDLE FileHandle,
  IN ACCESS_MASK DesiredAccess,
  IN POBJECT_ATTRIBUTES ObjectAttributes,
  OUT PIO_STATUS_BLOCK IoStatusBlock,
  IN PLARGE_INTEGER AllocationSize  OPTIONAL,
  IN ULONG FileAttributes,
  IN ULONG ShareAccess,
  IN ULONG CreateDisposition,
  IN ULONG CreateOptions,
  IN PVOID EaBuffer  OPTIONAL,
  IN ULONG EaLength)
  {
  //初始化此函数的返回值
  NTSTATUS rc=0;
  //定义一个数组用来保存当前用户操作的盘符
  char Dir[7]={'\0'};
 int Flag=0;
  ANSI_STRING ansiDirName;
  PUNICODE_STRING uniFileName;
  PWSTR pTemp = (PWSTR)ExAllocatePool( NonPagedPool, 256);
  uniFileName = (PUNICODE_STRING)ExAllocatePool( NonPagedPool, sizeof( UNICODE_STRING));
  uniFileName->Buffer = pTemp;
  //将用户操作的路径保存在变量ansiDirName中
  RtlUnicodeStringToAnsiString( &ansiDirName, ObjectAttributes->ObjectName, TRUE);
  memset(Dir,0,7);
  //将用户操作的盘符保存在数组Dir中
  memcpy(Dir,ansiDirName.Buffer,6);
  //比较和我们保护的盘符是否匹配 注意这里只是简单的使用"C:\"来表示C盘。实际的盘符应该是"\\??\\C:\\"
  if (strcmp(Dir,"C:\") ==0 ){
   //下面的参数设置是为了给系统返回一个系统级错误，这样系统就不会在这个盘符上创建文件。
       IoStatusBlock->Information=FILE_DOES_NOT_EXIST;
       rc=RealZwCreateFile(0,DesiredAccess,ObjectAttributes,IoStatusBlock,AllocationSize,FileAttributes,ShareAccess,CreateDisposition,CreateOptions,EaBuffer,EaLength);
       return rc;
  }
  //如果这个盘符不是我们保护的盘符，那么我们调用正常的ZwCreateFile函数来实现创建文件的过程
  rc=RealZwCreateFile(FileHandle,DesiredAccess,ObjectAttributes,IoStatusBlock,AllocationSize,FileAttributes,ShareAccess,CreateDisposition,            CreateOptions,EaBuffer,EaLength);
  return rc;
}
```