用WTSQuerySessionInformation函数，可以取得很多很多有用的东西.
```c++
    DWORD dwSessionID     =     -1; // 0 is 1st console session created on XP, 1 is 1st console session on Vista 
    LPTSTR pData          =     NULL; 
    DWORD cbReturned      =     0; 
    bool fActiveSession   =     false; 
    //    取ip, ok
    if( WTSQuerySessionInformation(WTS_CURRENT_SERVER_HANDLE, dwSessionID, /*WTSConnectState*/ /*WTSUserName*/WTSClientAddress, &pData, &cbReturned) 
        && (cbReturned == sizeof(INT)) ) { 
            // if we get WTSActive we're in the active session, otherwise we assume we're not in the active session (WTSDisconnected)
            fActiveSession = (*((INT *)pData) == WTSActive) ? true : false; 
    }
    PWTS_CLIENT_ADDRESS pWTSCA = (PWTS_CLIENT_ADDRESS)pData;    char address[100]= {0};
    sprintf(address, "address:%d.%d.%d.%d",pWTSCA->Address[2],pWTSCA->Address[3],pWTSCA->Address[4],pWTSCA->Address[5]); 
    MessageBoxA(NULL, address, "WTSClientAddress", MB_OK);
```
 判断是否远程登录  
```c++
GetSystemMetrics(SM_REMOTESESSION);
```
 