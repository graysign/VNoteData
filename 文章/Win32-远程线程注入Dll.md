```c++

BOOL LoadRemoteDll(DWORD dwProcessId, LPTSTR lpszLibName)
{    
	BOOL   bResult            = FALSE;    
	HANDLE hProcess           = NULL;    
	HANDLE hThread            = NULL;    
	PSTR   pszLibFileRemote   = NULL;    
	DWORD cch;    
	PTHREAD_START_ROUTINE pfnThreadRrn;    
	__try    {        
		//获得想要注入代码的进程的句柄        
		hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId);        
		if (NULL == hProcess)            
			__leave;        
		//计算DLL路径名需要的字节数        
		cch = 2 * (1 + lstrlen(lpszLibName));        
		//在远程线程中为路径名分配空间        
		pszLibFileRemote = (PSTR)VirtualAllocEx(hProcess, NULL, cch, MEM_COMMIT, PAGE_READWRITE);                
		if (pszLibFileRemote == NULL)            
			__leave;        
		//将DLL的路径名复制到远程进程的地址空间        
		if (!WriteProcessMemory(hProcess, (PVOID)pszLibFileRemote, (PVOID)lpszLibName, cch, NULL))            
			__leave;        
		//获得LoadLibraryA在Kernel.dll中得真正地址        
		pfnThreadRrn = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(TEXT("Kernel32")), "LoadLibraryW");        
		if (pfnThreadRrn == NULL)            
			__leave;        
		hThread = CreateRemoteThread(hProcess, NULL, 0, pfnThreadRrn, (PVOID)pszLibFileRemote, 0, NULL);        
		if (hThread == NULL)            
			__leave;        
		//等待远程线程终止        
		WaitForSingleObject(hThread, INFINITE);        
		bResult = TRUE;    
	}  __finally {        
		//关闭句柄        
		if (pszLibFileRemote != NULL)            
			VirtualFreeEx(hProcess, (PVOID)pszLibFileRemote, 0, MEM_RELEASE);        
		if (hThread != NULL)            
			CloseHandle(hThread);        
		if (hProcess != NULL)            
			CloseHandle(hProcess);    
	}
		return bResult;
}

BOOL GetProcessIdByName(LPWSTR szProcessName, LPDWORD lpPID)
{    
	//变量及其初始化    
	STARTUPINFO st;    
	PROCESS_INFORMATION pi;    
	PROCESSENTRY32 ps;    
	HANDLE hSnapshot;    
	ZeroMemory(&st, sizeof(STARTUPINFO));    
	ZeroMemory(&pi, sizeof(PROCESS_INFORMATION));    
	st.cb = sizeof(STARTUPINFO);    
	ZeroMemory(&ps, sizeof(PROCESSENTRY32));    
	ps.dwSize = sizeof(PROCESSENTRY32);    
	//遍历进程    
	hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);    
	if (hSnapshot == INVALID_HANDLE_VALUE)        
		return FALSE;    
	if (!Process32First(hSnapshot, &ps))        
		return FALSE;    
	do{        
		//比较进程名        
		if (lstrcmpi(ps.szExeFile, TEXT("calc.exe")) == 0){            
			//找到了            
			*lpPID = ps.th32ProcessID;            
			CloseHandle(hSnapshot);            
			return TRUE;        
		}    
	}while (Process32Next(hSnapshot, &ps));    
	//没有找到    
	CloseHandle(hSnapshot);    
	return FALSE;
}

//修改进程权限
BOOL EnablePrivilege(LPWSTR name)
{    
	HANDLE hToken;    
	BOOL rv;    
	TOKEN_PRIVILEGES priv = {1, {0, 0, SE_PRIVILEGE_ENABLED}};    
	LookupPrivilegeValue(0, name, &priv.Privileges[0].Luid);    
	OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES, &hToken);    
	AdjustTokenPrivileges(hToken, FALSE, &priv, sizeof priv, 0, 0);    
	rv = GetLastError() == ERROR_SUCCESS;    CloseHandle(hToken);    
	return rv;
}

int WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow)
{    
	DWORD dwPID;   
	//提权，获取SE_DEBUG_NAME 权限    
	//可以在其他进程的内存空间中写入，创建线程     
	if (0 == EnablePrivilege(SE_DEBUG_NAME))         
		return 0;    
	if (!GetProcessIdByName(TEXT("calc.exe"), &dwPID))        
		return 0;    
	//通过上传远程线程加载dll    
	//将msg.dll放置在系统目录下    
	if (!LoadRemoteDll(dwPID, TEXT("msg.dll")))        
		return 0;    
	return 1;
}
```  

创建远程线程句柄有下面几个步骤：  
1. 获得想要注入代码的进程的句柄 `hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId);`
2. 计算DLL路径名需要的字节数 `cch = 2 * (1 + lstrlen(lpszLibName));` 这里自己对宽字符的处理没有做好 ,可以使用wcslen函数
3. 在远程线程中为路径名分配空间 `pszLibFileRemote = (PSTR)VirtualAllocEx(hProcess, NULL, cch, MEM_COMMIT, PAGE_READWRITE);`
4. 将DLL的路径名复制到远程进程的地址空间 `WriteProcessMemory(hProcess, (PVOID)pszLibFileRemote, (PVOID)lpszLibName, cch, NULL);`
5. 获得LoadLibraryA在Kernel.dll中得真正地址 `pfnThreadRrn = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(TEXT("Kernel32")), "LoadLibraryW");`
6. 最后是最最关键的函数 `hThread = CreateRemoteThread(hProcess, NULL, 0, pfnThreadRrn, (PVOID)pszLibFileRemote, 0, NULL);`
7. 当然还有扫尾的工作  
```c++
//关闭句柄        
if (pszLibFileRemote != NULL)            
    VirtualFreeEx(hProcess, (PVOID)pszLibFileRemote, 0, MEM_RELEASE);        
if (hThread != NULL)            
    CloseHandle(hThread);        
if (hProcess != NULL)            
    CloseHandle(hProcess);
```

1. 关于进程令牌的代码，有时候不写也是可以通过的。这里还有待研究一下。
2. 被注入的程序有时候要放在相对目录下才会产生效果。关于这点自己还没有实践过。有待验证。
3. 像explorer.exe有时候会无法注入。换个简单的calc.exe完全可以。火狐，qq全部都可以。可能是explorer.exe有点特别。
4. 第一次远程注入可以成功，第二次就不行了，就不能弹出对话框了，貌似进程打开之后马上就关闭了。个人以为原因可能处在dll的fdwReason。有时间尝试着调试一下。
`自己改了一下，应该是这样的。上面的代码里面没有FreeLibrary，那么dll还是在远程地址空间里面。第二次想要注入的时候是不会发送DLL_PROCESS_ATTACH 消息的。
DLL_PROCESS_ATTACH`消息的发送是这样的：  
    线程调用LoadLibrary   --<   Dll是否已经被映射到了进程的地址空间中  (如果是)  --<   递增DLL的使用计数  --<  使用计数器是否等于1 （如果是） --<  调用DLL的DllMain并且传入DLL_PROCESS_ATTACH  
    
所以说，在Dll已经被映射到了进程地址空间之后，只不过DLL的使用计数增加了，如果是2，那么很明显就不会调用DLL的DllMain并且传入DLL_PROCESS_ATTACH
但是如果我把dll改一下，把弹框代码写在DLL_THREAD_ATTACH里面，那么每次创建线程的时候就都会收到DLL_THREAD_ATTACH消息了。