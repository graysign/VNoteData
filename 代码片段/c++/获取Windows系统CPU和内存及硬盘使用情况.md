1. 获取Windows系统内存使用率  
```c++
//windows 内存 使用率  
DWORD getWin_MemUsage(){  
    MEMORYSTATUS ms;  
    ::GlobalMemoryStatus(&ms);  
    return ms.dwMemoryLoad;  
}  
```
2. 获取windowsCPU使用率
```c++
__int64 CompareFileTime(FILETIME time1, FILETIME time2)  
{  
    __int64 a = time1.dwHighDateTime << 32 | time1.dwLowDateTime;  
    __int64 b = time2.dwHighDateTime << 32 | time2.dwLowDateTime;  
  
    return (b - a);  
}  
//WIN CPU使用情况  
void getWin_CpuUsage(){  
    HANDLE hEvent;  
    BOOL res;  
    FILETIME preidleTime;  
    FILETIME prekernelTime;  
    FILETIME preuserTime;  
    FILETIME idleTime;  
    FILETIME kernelTime;  
    FILETIME userTime;  
  
    res = GetSystemTimes(&idleTime, &kernelTime, &userTime);  
    preidleTime = idleTime;  
    prekernelTime = kernelTime;  
    preuserTime = userTime;  
  
    hEvent = CreateEventA(NULL, FALSE, FALSE, NULL); // 初始值为 nonsignaled ，并且每次触发后自动设置为nonsignaled  
  
    while (true){  
        WaitForSingleObject(hEvent, 1000);  
        res = GetSystemTimes(&idleTime, &kernelTime, &userTime);  
  
        __int64 idle = CompareFileTime(preidleTime, idleTime);  
        __int64 kernel = CompareFileTime(prekernelTime, kernelTime);  
        __int64 user = CompareFileTime(preuserTime, userTime);  
  
        __int64 cpu = (kernel + user - idle) * 100 / (kernel + user);  
        __int64 cpuidle = (idle)* 100 / (kernel + user);  
        cout << "CPU利用率:" << cpu << "%" << " CPU空闲率:" << cpuidle << "%" << endl;  
  
        preidleTime = idleTime;  
        prekernelTime = kernelTime;  
        preuserTime = userTime;  
    }  
}  
```
3. 获取 WIN 硬盘使用情况  
```c++
//获取 WIN 硬盘使用情况  
int getWin_DiskUsage(){  
    int DiskCount = 0;  
    DWORD DiskInfo = GetLogicalDrives();  
    //利用GetLogicalDrives()函数可以获取系统中逻辑驱动器的数量，函数返回的是一个32位无符号整型数据。    
    while (DiskInfo)//通过循环操作查看每一位数据是否为1，如果为1则磁盘为真,如果为0则磁盘不存在。    
    {  
        if (DiskInfo & 1)//通过位运算的逻辑与操作，判断是否为1    
        {  
            ++DiskCount;  
        }  
        DiskInfo = DiskInfo >> 1;//通过位运算的右移操作保证每循环一次所检查的位置向右移动一位。    
        //DiskInfo = DiskInfo/2;    
    }  
    cout << "Logical Disk Number:" << DiskCount << endl;  
    //-----------------------------------------------------------------------------------------  
    int DSLength = GetLogicalDriveStrings(0, NULL);  
    //通过GetLogicalDriveStrings()函数获取所有驱动器字符串信息长度。    
    char* DStr = new char[DSLength];//用获取的长度在堆区创建一个c风格的字符串数组    
    GetLogicalDriveStrings(DSLength, (LPTSTR)DStr);  
    //通过GetLogicalDriveStrings将字符串信息复制到堆区数组中,其中保存了所有驱动器的信息。    
  
    int DType;  
    int si = 0;  
    BOOL fResult;  
    unsigned _int64 i64FreeBytesToCaller;  
    unsigned _int64 i64TotalBytes;  
    unsigned _int64 i64FreeBytes;  
  
    for (int i = 0; i<DSLength / 4; ++i)//为了显示每个驱动器的状态，则通过循环输出实现，由于DStr内部保存的数据是A:\NULLB:\NULLC:\NULL，这样的信息，所以DSLength/4可以获得具体大循环范围    
    {  
        char dir[3] = { DStr[si], '：', '\\' };  
        cout << dir;  
        DType = GetDriveType(DStr + i * 4);  
        //GetDriveType函数，可以获取驱动器类型，参数为驱动器的根目录    
        if (DType == DRIVE_FIXED)  
        {  
            cout << "Hard Disk";  
        }  
        else if (DType == DRIVE_CDROM)  
        {  
            cout << "CD-ROM";  
        }  
        else if (DType == DRIVE_REMOVABLE)  
        {  
            cout << "Removable Disk";  
        }  
        else if (DType == DRIVE_REMOTE)  
        {  
            cout << "Network Disk";  
        }  
        else if (DType == DRIVE_RAMDISK)  
        {  
            cout << "Virtual RAM Disk";  
        }  
        else if (DType == DRIVE_UNKNOWN)  
        {  
            cout << "Unknown Device";  
        }  
  
        fResult = GetDiskFreeSpaceEx(  
            dir,  
            (PULARGE_INTEGER)&i64FreeBytesToCaller,  
            (PULARGE_INTEGER)&i64TotalBytes,  
            (PULARGE_INTEGER)&i64FreeBytes);  
        //GetDiskFreeSpaceEx函数，可以获取驱动器磁盘的空间状态,函数返回的是个BOOL类型数据    
        if (fResult)//通过返回的BOOL数据判断驱动器是否在工作状态    
        {  
            cout << " totalspace:" << (float)i64TotalBytes / 1024 / 1024 << " MB";//磁盘总容量    
            cout << " freespace:" << (float)i64FreeBytesToCaller / 1024 / 1024 << " MB";//磁盘剩余空间    
        }  
        else  
        {  
            cout << " 设备未准备好";  
        }  
        cout << endl;  
        si += 4;  
    }  
    return 0;  
}  

```