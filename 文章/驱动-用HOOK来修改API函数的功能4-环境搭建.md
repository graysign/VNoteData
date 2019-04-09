上面的3篇文章已经大概的讲述了HOOK API的编写方法，可是如何让它们真正的运行起来呢？  
这就需要搭建环境，而这个环境你必须使用DDK。我在这里假设你已经安装了DDK，并且会使用DDK来编译一个  
WDM驱动程序。这里我要说的是如何在代码中将我上面的3篇文章中讲到的功能串在一起，以及编写WDM驱动程序所需要的Sources文件。  
我的Sources文件是这样写的：  
```source
TARGETNAME=TestDriver
TARGETTYPE=DRIVER
TARGETPATH=obj
BROWSER_INFO=1
C_DEFINES=-DDRIVER
INCLUDES=c:\ntddk\inc;
USER_C_FLAGS=/FAcs
SOURCES=My.c
```
其中TARGETNAME=TestDriver指明编译出来的文件名称叫做TestDriver。  
TARGETTYPE=DRIVER指明生成的文件类型是*.sys。  
SOURCES=My.c指明需要编译的源代码文件。  
下面来看看My.c文件内容。  
作为一个WDM驱动，首先运行的应该是DriverEntry例程。该例程类似于C语言中的Main函数。  

```c++
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) 
{ 
 UNICODE_STRING nameString, linkString; 
 UNICODE_STRING *InstallDir=NULL;
 PDEVICE_OBJECT deviceObject; 
 NTSTATUS status; 
 WCHAR wBuffer[200]; 
 nameString.Buffer = wBuffer; 
 nameString.MaximumLength = 200; 
 
 //DriverUnload例程用来处理驱动卸载时所做的恢复工作.
 DriverObject->DriverUnload = DriverUnload; 
 RtlInitUnicodeString(&nameString, L"\\Device\\MyDriver"); 
 status = IoCreateDevice(DriverObject, 0,&nameString, FILE_DEVICE_UNKNOWN, 0, TRUE, &deviceObject ); 
 if (!NT_SUCCESS( status )) 
 {
  return status; 
 }
 deviceObject->Flags |= DO_BUFFERED_IO;
 RtlInitUnicodeString(&linkString, L"\\??\\MyDriver");
 status = IoCreateSymbolicLink (&linkString, &nameString); 
 if (!NT_SUCCESS( status )) 
 { 
  IoDeleteDevice (DriverObject->DeviceObject); 
  return status; 
 } 
 
 //MydrvDispatch例程是一个IRP的分发例程（在我的代码中没有怎么用到它）
 DriverObject->MajorFunction[IRP_MJ_CREATE] = MydrvDispatch; 
 DriverObject->MajorFunction[IRP_MJ_CLOSE] = MydrvDispatch;
 DriverObject->MajorFunction[IRP_MJ_CLEANUP] = MydrvDispatch;
 
 //MydrvDispatchIoctl例程比较重要，该例程可以接受来自外界EXE程序发送给它的参数。例如在我的项目中，由一个外界exe程序
 //发送过来需要保护的文件名称、注册表键值等数据，都是使用这个例程来接收的。
 
 DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = MydrvDispatchIoctl;
 __asm{
  mov eax, cr0 
  mov CR0VALUE, eax 
  and eax, 0fffeffffh 
  mov cr0, eax 
  }
 //看过我文章的朋友一定不会对下面的代码陌生。
 
  //文件删除
  RealZwSetInformationFile=(ZWSETINFORMATIONFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetInformationFile")]);
  (ZWSETINFORMATIONFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetInformationFile")])=HookZwSetInformationFile;
  //注册表删除
  RealZwDeleteKey=(REALZWDELETEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteKey")]);
  (REALZWDELETEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteKey")])=HookZwDeleteKey;
  //删除注册表内容
  RealZwDeleteValueKey=(REALZWDELETEVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteValueKey")]);
  (REALZWDELETEVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteValueKey")])=HookZwDeleteValueKey;
  //设置注册表键值
  RealZwSetValueKey=(REALZWSETVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetValueKey")]);
  (REALZWSETVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetValueKey")])=HookZwSetValueKey;
  //创建文件
  RealZwCreateFile=(REALZWCREATEFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwCreateFile")]);
  (REALZWCREATEFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwCreateFile")])=HookZwCreateFile;
  
 __asm{
  mov eax, CR0VALUE 
  mov cr0, eax 
  }
 return STATUS_SUCCESS; 
}

//在卸载驱动的时候使用的DriverUnload例程代码如下：
VOID DriverUnload (IN PDRIVER_OBJECT pDriverObject) 
{ 
 UNICODE_STRING nameString; 
 RtlInitUnicodeString(&nameString, L"\\??\\MyDriver"); 
 IoDeleteSymbolicLink(&nameString);
 IoDeleteDevice(pDriverObject->DeviceObject);
 //卸载设置文件属性
 (ZWSETINFORMATIONFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetInformationFile")])=RealZwSetInformationFile;
 //删除注册表键值
 (REALZWDELETEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteKey")])=RealZwDeleteKey;
 //删除注册表内容
 (REALZWDELETEVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwDeleteValueKey")])=RealZwDeleteValueKey;
 //设置注册表键值
 (REALZWSETVALUEKEY)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwSetValueKey")])=RealZwSetValueKey;
 //创建文件
 (REALZWCREATEFILE)(KeServiceDescriptorTable->ServiceTableBase[SYSTEMSERVICE("ZwCreateFile")])=RealZwCreateFile;
 return; 
}
```

这样就搭建出了一个完整的WDM驱动框架。编译后可以生成一个具有一定功能的驱动程序。  
下次我将会写如何将我们编写好的驱动进行安装，并通过外界的EXE程序和这个驱动程序进行通信。