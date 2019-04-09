今天我写一写如何使用HOOK的方法来保护一些特定的文件不被删除。  
    在"未文档化函数中"有个函数叫做`ZwSetInformationFile`。这个函数对应的WIN32的函数有"`SetFileAttributes、SetEndOfFile、SetFilePointer、SetFileTime、DeleteFile`"。也就是说，以上的函数均是和这个`ZwSetInformationFile`函数有关。如果我们截获到这个函数那么我们就可以做到禁止删除文件。  
    下面是我的代码。我首先对用户操作的文件名和我们要保护的文件名进行比较，如果文件名相同，则继续判断路径是否相同。如果相同，那么这个文件就是我们保护的文件。否则继续用户的原有操作。  

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
typedef struct _FILE_NAMES_INFORMATION {
    ULONG NextEntryOffset;
    ULONG FileIndex;
    ULONG FileNameLength;
    WCHAR FileName[1];
} FILE_NAMES_INFORMATION, *PFILE_NAMES_INFORMATION;
...
//反删除文件需要的函数
//(1).声明原有函数
typedef NTSTATUS (*ZWSETINFORMATIONFILE)(
 IN HANDLE  FileHandle,
 OUT PIO_STATUS_BLOCK  IoStatusBlock,
 IN PVOID  FileInformation,
 IN ULONG  Length,
 IN FILE_INFORMATION_CLASS  FileInformationClass
 );
ZWSETINFORMATIONFILE RealZwSetInformationFile;
 
//(2).定义HOOK反删除文件的函数
NTSTATUS HookZwSetInformationFile(
 IN HANDLE  FileHandle,
 OUT PIO_STATUS_BLOCK  IoStatusBlock,
 IN PVOID  FileInformation,
 IN ULONG  Length,
 IN FILE_INFORMATION_CLASS  FileInformationClass
 );
...
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) 
{
...... 
RealZwSetInformationFile=(ZWSETINFORMATIONFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetInformationFile")]);
  (ZWSETINFORMATIONFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetInformationFile")])=HookZwSetInformationFile;
//上面两行的代码含义请参照我上面的文章。
}

NTSTATUS HookZwSetInformationFile(
  IN HANDLE  FileHandle,
  OUT PIO_STATUS_BLOCK  IoStatusBlock,
  IN PVOID  FileInformation,
  IN ULONG  Length,
  IN FILE_INFORMATION_CLASS  FileInformationClass
)
{
 PVOID *pFileObject;
 NTSTATUS rc;
 ULONG ActualLength;
 int DelInt;
 PUNICODE_STRING puniFileName_Del;
 ANSI_STRING ansiFileName_Del,
    m_ansiFileName_Del,
    ansiGetFileName_Del,
    m_ansiGetFileName_Del;
 IO_STATUS_BLOCK MyIoStatusBlock;
 PFILE_NAMES_INFORMATION pFileInfo; 
 char DeleteFileDir[MaxBuf]={'\0'};
 char TempDel[2]={'\0'};
 char TempInstallDir[256]={'\0'};//定义临时使用的安装路径数组
 UNICODE_STRING pHideFileDir; //定义反删除读取文件的文件名
 UNICODE_STRING *pFullPath=NULL;
 int Judge=0;
 int CompareJudge=0;
 int i;
 pFileInfo = (PFILE_NAME_INFORMATION)ExAllocatePool( NonPagedPool, sizeof(FILE_NAME_INFORMATION)*255);
 //调用ZwQueryInformationFile函数将操作文件的信息放如pFileInfo。
 rc = ZwQueryInformationFile( FileHandle,&MyIoStatusBlock,pFileInfo,sizeof(FILE_NAME_INFORMATION)*255,FileNameInformation );
 if(NT_SUCCESS(rc))
 {
  //定义变量
  PWSTR pTemp = (PWSTR)ExAllocatePool( NonPagedPool, sizeof(PWSTR)*pFileInfo->FileNameLength);
  puniFileName_Del = (PUNICODE_STRING)ExAllocatePool( NonPagedPool, sizeof( UNICODE_STRING));
  puniFileName_Del->Buffer = pTemp;
  //将文件名成拷贝到puniFileName_Del变量中
  RtlCopyMemory( puniFileName_Del->Buffer, pFileInfo->FileName, pFileInfo->FileNameLength);
  puniFileName_Del->Length = (USHORT)pFileInfo->FileNameLength;
  puniFileName_Del->MaximumLength = (USHORT)pFileInfo->FileNameLength;
  
  //对文件名称转换大小写
  RtlUnicodeStringToAnsiString( &ansiFileName_Del, puniFileName_Del, TRUE);
  RtlUnicodeStringToAnsiString( &m_ansiFileName_Del, puniFileName_Del, TRUE);
  RtlUpperString(&ansiFileName_Del,&m_ansiFileName_Del);
  //将文件名称保存在变量ansiFileName_Del中
  RtlFreeAnsiString(&m_ansiFileName_Del);
  //比较我们要保护的文件名称是否和正在操作的文件名称相同
  if (RtlCompareMemory(ansiFileName_Del.Buffer,"Test1.txt",strlen("Test1.txt") ) == strlen("Test1.txt"))
  {
    rc = ObReferenceObjectByHandle(FileHandle,0,NULL,KernelMode,(PVOID*)&pFileObject,NULL);
    if(NT_SUCCESS(rc))   
    {
     pFullPath = (UNICODE_STRING *)ExAllocatePool(NonPagedPool,MaxBuf);
     RtlZeroMemory(pFullPath,MaxBuf);
     pFullPath->MaximumLength = MaxBuf ;
     //得到文件的全路径
     rc = ObQueryNameString(pFileObject,pFullPath,MaxBuf,&ActualLength);
     if(rc != STATUS_SUCCESS)
     {
      //如果失败 则调用正常处理
      rc=RealZwSetInformationFile(FileHandle,IoStatusBlock,FileInformation,Length,FileInformationClass);
      RtlFreeAnsiString(&ansiFileName_Del);
      ExFreePool(puniFileName_Del);
      ExFreePool(pTemp);
      ExFreePool(pFileInfo);
      ExFreePool(pFullPath);
      ObDereferenceObject(pFileObject);
      return rc;
     }
     ObDereferenceObject(pFileObject);
     //将文件的全路径转换成大写
     RtlUnicodeStringToAnsiString(&ansiGetFileName_Del,pFullPath,TRUE);
     RtlUnicodeStringToAnsiString(&m_ansiGetFileName_Del,pFullPath,TRUE);
     RtlUpperString(&ansiGetFileName_Del,&m_ansiGetFileName_Del);
     RtlFreeAnsiString(&m_ansiGetFileName_Del);
     memset(DeleteFileDir,0,MaxBuf);
     memcpy(DeleteFileDir,ansiGetFileName_Del.Buffer,ansiGetFileName_Del.Length);
     RtlFreeAnsiString(&ansiGetFileName_Del);
     ExFreePool(pFullPath);
     //判断文件的全路径是否和我们需要保护的文件全路径匹配
     if(RtlCompareMemory(DeleteFileDir,"\\??\\C:\\Test1.txt",strlen("\\??\\C:\\Test1.txt") ) == strlen("\\??\\C:\\Test1.txt"))
     {
      //如果匹配，直接返回，不调用删除功能
      RtlFreeAnsiString(&ansiFileName_Del);
      ExFreePool(puniFileName_Del);
      ExFreePool(pTemp);
      ExFreePool(pFileInfo);
      return(rc);
     } 
    } 
    else
    {
     //如果调用ObReferenceObjectByHandle函数不正常，则调用原来的操作
     rc=RealZwSetInformationFile(FileHandle,IoStatusBlock,FileInformation,Length,FileInformationClass);
     RtlFreeAnsiString(&ansiFileName_Del);
     ExFreePool(puniFileName_Del);
     ExFreePool(pTemp);
     ExFreePool(pFileInfo);
     return rc;
    }
   }
  }
  //如果文件名称不匹配，则调用原来的操作
  rc=RealZwSetInformationFile(FileHandle,IoStatusBlock,FileInformation,Length,FileInformationClass);
  RtlFreeAnsiString(&ansiFileName_Del);
  ExFreePool(puniFileName_Del);
  ExFreePool(pTemp);
 }
 ExFreePool(pFileInfo);
 return(rc);
}
```

以上就是我编写的保护特定的文件不被删除的代码。  
为了方便大家的使用，这次我将"未文档化函数"的说明上传上去。