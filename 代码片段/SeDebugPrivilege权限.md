# SeDebugPrivilege权限
```c++
　  HANDLE hToken;
　　LUID DebugNameValue;
　　TOKEN_PRIVILEGES Privileges;
　　DWORD dwRet;

　　OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken);
　　LookupPrivilegeValue(NULL, "SeDebugPrivilege", &DebugNameValue);
　　Privileges.PrivilegeCount=1;
　　Privileges.Privileges[0].Luid=DebugNameValue;
　　Privileges.Privileges[0].Attributes=SE_PRIVILEGE_ENABLED;
　　AdjustTokenPrivileges(hToken, FALSE,&Privileges, sizeof(Privileges), NULL, &dwRet);
　　CloseHandle(hToken);
```