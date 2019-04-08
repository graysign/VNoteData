原始套接字是WINSOCK公开的一个套接字编程接口，它让我们可以在 IP 层对套接字进行编程，控制其行为，常见的应用有抓包 (Sniffer)、分析包、洪水攻击、  
ICMP ping等，但它不能截取包（所谓的截取包就是把包拦截下来，要做到这种“防火墙”的功能，还需要再低一层的驱动层才可以做到）。但是能把网络上的  
包复制到本机就已经是一个很有用的功能了。要写原始套接字的程序其实也很容易，因为WINDOWS已经帮我们定义实现好了这些接口（WINSOCK）。另外我们还要  
有一些定义，就是IP包头、UDP包头等的那些
结构定义，具体请查看下面的代码。
下面是我定义的一个RawSniffer类，这个类是使用了原始套接字来实现一个侦听器，这个类还依赖于MFC，下面是类的代码：    
```MFC
// 头文件
#ifndef RAW_DEF_H
#define RAW_DEF_H
#include <winsock2.h> 
#pragma comment(lib,"ws2_32")
#include <ws2tcpip.h>
#include <mstcpip.h> // 此文件是 Windows platform SDK 的函数，如果找不到，请安装SDK
#define PROTOCOL_STRING_ICMP_TXT "ICMP"
#define PROTOCOL_STRING_TCP_TXT "TCP"
#define PROTOCOL_STRING_UDP_TXT "UDP"
#define PROTOCOL_STRING_SPX_TXT "SPX"
#define PROTOCOL_STRING_NCP_TXT "NCP"
#define PROTOCOL_STRING_UNKNOW_TXT "UNKNOW"
// 定义IP首部 
typedef struct ip_hdr 
{ 
 unsigned char h_verlen;   // 4位首部长度,4位IP版本号 
 unsigned char tos;    // 8位服务类型TOS 
 unsigned short total_len;       // 16位总长度（字节） 
 unsigned short ident;      // 16位标识 
 unsigned short frag_and_flags;  // 3位标志位 
 unsigned char ttl;       // 8位生存时间 TTL 
 unsigned char proto;      // 8位协议 (TCP, UDP 或其他) 
 unsigned short checksum;  // 16位IP首部校验和 
 unsigned int sourceIP;   // 32位源IP地址 
 unsigned int destIP;   // 32位目的IP地址 
}IPHEADER;
// 定义TCP伪首部 
typedef struct tsd_hdr 
{ 
 unsigned long saddr;    // 源地址 
 unsigned long daddr;    // 目的地址 
 char mbz;                        // 0
 char ptcl;       // 协议类型 UDP的协议类型为17，TCP为6 
 unsigned short tcpl;    // TCP数据包长度 
}PSDHEADER;
// 定义TCP首部 
typedef struct tcp_hdr 
{ 
 USHORT th_sport;     // 16位源端口 
 USHORT th_dport;     // 16位目的端口 
 unsigned int th_seq;    // 32位序列号 
 unsigned int th_ack;    // 32位确认号 
 unsigned char th_lenres;   // 4位首部长度/6位保留字 
 unsigned char th_flag;    // 6位标志位 
 USHORT th_win;      // 16位窗口大小 
 USHORT th_sum;      // 16位校验和 
 USHORT th_urp;      // 16位紧急数据偏移量 
}TCPHEADER;
// 定义ICMP首部
typedef struct icmp_hdr
{
 unsigned char  i_type;           // 类型
 unsigned char  i_code;           // 代码
 unsigned short i_cksum;          // 校验码
 unsigned short i_id;             // 非标准的ICMP首部  
 unsigned short i_seq;
 unsigned long  timestamp;
}ICMPHEADER;
// 定义UDP首部
// The UDP packet is lick this. Took from RFC768.
//                  0      7 8     15 16    23 24    31  
//                 +--------+--------+--------+--------+ 
//                 |     Source      |   Destination   | 
//                 |      Port       |      Port       | 
//                 +--------+--------+--------+--------+ 
//                 |                 |                 | 
//                 |     Length      |    Checksum     | 
//                 +--------+--------+--------+--------+ 
//                 |                                     
//                 |          data octets ...            
//                 +---------------- ...      
typedef struct udp_hdr  // 8 Bytes
{
 unsigned short uh_sport;         
 unsigned short uh_dport;
 unsigned short uh_len;
 unsigned short uh_sum;
} UDPHEADER;

/* 
// 函数实现不要放在头文件，否则会导致在不同的地方重复定义
//CheckSum:计算校验和的子函数 
USHORT checksum(USHORT *buffer, int size) 
{ 
 unsigned long cksum=0; 
 while(size >1) 
 { 
  cksum+=*buffer++; 
  size -=sizeof(USHORT); 
 } 
 if(size ) 
 { 
  cksum += *(UCHAR*)buffer; 
 } 
 
 cksum = (cksum >> 16) + (cksum & 0xffff); 
 cksum += (cksum >>16); 
 return (USHORT)(~cksum); 
}
*/
USHORT checksum(USHORT *buffer, int size);

// 回调函数
// 抓到一个包就会调用这个回调函数
typedef int (CALLBACK *CaptureDef)(CString  &strMsg);
class YRawSniffer
{
public:
 YRawSniffer();
 ~YRawSniffer();
 BOOL StartAll();
 BOOL ExitAll();
 BOOL Capture(CaptureDef CaptureFunc = NULL);
 BOOL StopCapture();
 static DWORD WINAPI CaptureThread(LPVOID lpParam);
 HANDLE m_hCaptureThread;
 // Filter 过滤条件
 BOOL m_bCapTCP;
 BOOL m_bCapUDP;
 BOOL m_bCapICMP;
 CString m_strSrcIP;
 CString m_strDstIP;
 
 SOCKET m_rawSock;
// CString m_strFilePath;
private:
 CaptureDef m_CaptureFunc;
 BOOL m_bExitCapture;
 
 IPHEADER m_ipHeader; 
 TCPHEADER m_tcpHeader;
 ICMPHEADER m_icmpheader;
 UDPHEADER m_udpheader;
};
#endif
// CPP 文件
#include "stdafx.h"
#include "RawDef.h"
//CheckSum:计算校验和的子函数 
USHORT checksum(USHORT *buffer, int size) 
{ 
 unsigned long cksum=0; 
 while(size >1) 
 { 
  cksum+=*buffer++; 
  size -=sizeof(USHORT); 
 } 
 if(size ) 
 { 
  cksum += *(UCHAR*)buffer; 
 } 
 
 cksum = (cksum >> 16) + (cksum & 0xffff); 
 cksum += (cksum >>16); 
 return (USHORT)(~cksum); 
}
char * GetProtocol(int proto)
{
 switch(proto)
 {
     case IPPROTO_ICMP: return PROTOCOL_STRING_ICMP_TXT;
  case IPPROTO_TCP:  return PROTOCOL_STRING_TCP_TXT;
  case IPPROTO_UDP:  return PROTOCOL_STRING_UDP_TXT;
  default:     return PROTOCOL_STRING_UNKNOW_TXT;
 }
}
DWORD WaitForObjectEx( HANDLE hHandle, DWORD dwMilliseconds )
{
 BOOL bRet;
 MSG msg;
 INT iWaitRet;
 DWORD nTimeOut = 0;
 while( (bRet = ::GetMessage( &msg, NULL, 0, 0 )) != 0)
 { 
  if( nTimeOut++ * 100 >= dwMilliseconds )
   break;
  iWaitRet = WaitForSingleObject(hHandle, 100);
  if(iWaitRet != WAIT_TIMEOUT)
  {
   break;
  }
  if (bRet == -1)
  {
   break;
  }
  else
  {
   ::TranslateMessage(&msg); 
   ::DispatchMessage(&msg); 
  }
 }
 return iWaitRet;
}
 

///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
YRawSniffer *g_rawSniffer;
YRawSniffer::YRawSniffer()
{
 m_bCapTCP = FALSE;
 m_bCapUDP = FALSE;
 m_bCapICMP = FALSE;
 m_strSrcIP = "";
 m_strDstIP = "";
 m_rawSock = INVALID_SOCKET;
// m_strFilePath = "";
 m_bExitCapture = FALSE;
 m_CaptureFunc = NULL;
 m_hCaptureThread = NULL;
 g_rawSniffer = this;
 WSADATA wsaData;
 if(WSAStartup(MAKEWORD(2, 2), &wsaData)!=0)
 {
  TRACE1("WSAStartup() ERROR! %d", GetLastError());
  return;
 }
}
YRawSniffer::~YRawSniffer()
{
 WSACleanup(); 
}
BOOL YRawSniffer::ExitAll()
{
 m_bExitCapture = TRUE;
 shutdown(m_rawSock, SD_BOTH);
 
 if(m_hCaptureThread != NULL)
 {
  DWORD dwRet = 0;
  dwRet = WaitForObjectEx(m_hCaptureThread, INFINITE);
  if(dwRet == WAIT_OBJECT_0)
  {
   TRACE("CaptureThread exit Success!");
  }
  closesocket(m_rawSock);
  m_rawSock = INVALID_SOCKET;
  CloseHandle(m_hCaptureThread);
  m_hCaptureThread = NULL;
 }
 TRACE("ExitAll OK!");
 return TRUE;
}
BOOL YRawSniffer::StartAll()
{
 m_bExitCapture = FALSE;
 SOCKADDR_IN addr_in;
 if(m_rawSock == INVALID_SOCKET)
  m_rawSock = socket(AF_INET, SOCK_RAW, IPPROTO_IP);
    BOOL flag = TRUE;
 if(setsockopt(m_rawSock, IPPROTO_IP, IP_HDRINCL, (char*)&flag, sizeof(flag)) != 0)
 {
        TRACE1("setsockopt() ERROR! %d", WSAGetLastError());
  return FALSE;
 }
 
 char  LocalName[16];
 struct hostent *pHost;
 // 获取本机名 
 if (gethostname((char*)LocalName, sizeof(LocalName)-1) == SOCKET_ERROR) 
 {
  TRACE1("gethostname error! %d", WSAGetLastError());
  return FALSE;
 }
 
 // 获取本地 IP 地址 
 if ((pHost = gethostbyname((char*)LocalName)) == NULL) 
 {
  TRACE1("gethostbyname error! %d", WSAGetLastError());
  return FALSE;
 }
 
 // m_strSrcIP = pHost->h_addr_list[0];
 addr_in.sin_addr  = *(in_addr *)pHost->h_addr_list[0]; // IP 
 addr_in.sin_family = AF_INET; 
 addr_in.sin_port  = htons(57274);
 
 if( bind(m_rawSock, (struct sockaddr *)&addr_in, sizeof(addr_in)) != 0)
 {
  TRACE1("bind error! %d", WSAGetLastError());
  return FALSE;
 }
 
 // 设置网卡的I/O行为，接收网络上所有的数据包
 DWORD dwValue = 1; 
 if( ioctlsocket(m_rawSock, SIO_RCVALL, &dwValue) != 0)
 {
  TRACE1("ioctlsocket error! %d", WSAGetLastError());
  return FALSE;
 } 
 return TRUE;
}
DWORD WINAPI YRawSniffer::CaptureThread(LPVOID lpParam)
{
 //CFile fLog;
 //BOOL bLogFile = FALSE;
 // 打开记录文件
 //if(g_rawSniffer->m_strFilePath == "")
 // g_rawSniffer->m_strFilePath = "c://Capture.txt";
 //if(g_rawSniffer->m_strFilePath != "")
// {
//  if( !fLog.Open(g_rawSniffer->m_strFilePath, CFile::modeCreate|CFile::modeReadWrite) )
//   TRACE1("file fLog Open failed! %d", GetLastError());
//  else
//   bLogFile = TRUE;
// }
 const int MAX_RECEIVEBUF = 1000;
 char recvBuf[MAX_RECEIVEBUF] = {0};
 char msg[MAX_RECEIVEBUF] = {0};
 char *ptr = NULL;
 CString strLog, strTmp, strContent;
 DWORD nTCPCnt = 0, nUDPCnt = 0, nICMPCnt = 0;
 while(!g_rawSniffer->m_bExitCapture)
 {
  int ret = recv(g_rawSniffer->m_rawSock, recvBuf, MAX_RECEIVEBUF, 0);
  if(ret == SOCKET_ERROR)
   TRACE1("%d, recv(g_rawSniffer->m_rawSock, recvBuf, MAX_RECEIVEBUF, 0) failed!", GetLastError());
  
  strLog = "";
  strContent = "";
  if(ret > 0)
  {   
   g_rawSniffer->m_ipHeader = *(IPHEADER*)recvBuf;
            
   // 取得正确的IP头长度
   int iIphLen = sizeof(unsigned long) * (g_rawSniffer->m_ipHeader.h_verlen & 0xf);
   int cpysize = 0;
   // 过滤目标IP或者源IP
   //if(g_rawSniffer->m_strSrcIP.Find(".") > 0 
   // || g_rawSniffer->m_strDstIP.Find(".") > 0)
   {
    if(g_rawSniffer->m_strSrcIP != "" 
     || g_rawSniffer->m_strDstIP != "")
    {
     if( inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.sourceIP) != g_rawSniffer->m_strSrcIP
      && inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.destIP) != g_rawSniffer->m_strDstIP)
      continue;
    }
   }
   /*
   // 过滤目标IP或者源IP
   if(g_rawSniffer->m_strSrcIP != "")
   {
    if( inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.sourceIP) != g_rawSniffer->m_strSrcIP)
     continue;
   }
   if(g_rawSniffer->m_strDstIP != "")
   {
    if( inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.destIP) != g_rawSniffer->m_strDstIP)
     continue;
   }
   */
   
   if(g_rawSniffer->m_ipHeader.proto == IPPROTO_TCP && g_rawSniffer->m_bCapTCP)
   {
    nTCPCnt++;
    g_rawSniffer->m_tcpHeader = *(TCPHEADER*)(recvBuf + iIphLen);
    strTmp.Format("取得 %d TCP包", nTCPCnt); strLog += strTmp;
    strTmp.Format("协议： %s /r/n", GetProtocol(g_rawSniffer->m_ipHeader.proto)); strLog += strTmp;
    strTmp.Format("IP源地址： %s /r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.sourceIP)); strLog += strTmp;
    strTmp.Format("IP目标地址: %s /r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.destIP)); strLog += strTmp;
    strTmp.Format("TCP源端口号： %d /r/n", g_rawSniffer->m_tcpHeader.th_sport); strLog += strTmp;
    strTmp.Format("TCP目标端口号：%d /r/n", g_rawSniffer->m_tcpHeader.th_dport); strLog += strTmp;
    strTmp.Format("数据包长度： %d /r/n", ntohs(g_rawSniffer->m_ipHeader.total_len)); strLog += strTmp;
    strTmp.Format("TCP数据包的报文内容：/r/n"); strLog += strTmp;
    
    ptr = recvBuf + iIphLen + (4 * ((g_rawSniffer->m_tcpHeader.th_lenres & 0xf0)>>4|0));
    cpysize = ntohs(g_rawSniffer->m_ipHeader.total_len) - (iIphLen + (4 * ((g_rawSniffer->m_tcpHeader.th_lenres & 0xf0)>>4|0)));
    
    // ASCII码
    memcpy(msg, ptr, cpysize);
    for(int i = 0; i < cpysize ; i++)
    {
     if(msg[i] >= 32 && msg[i] < 255)
     {
      strContent.Format("%c", (unsigned char)msg[i]); strLog += strContent;
     }
     else
     {
      strContent.Format("."); strLog += strContent;
     }
    }
    strTmp.Format("/r/n /r/n"); strLog += strTmp;
   }
   
   
   if(g_rawSniffer->m_ipHeader.proto == IPPROTO_ICMP  && g_rawSniffer->m_bCapICMP)
   {
    nICMPCnt++;
    g_rawSniffer->m_icmpheader = *(ICMPHEADER*)(recvBuf + iIphLen);
    strTmp.Format("取得 %d ICMP包", nICMPCnt); strLog += strTmp;
    strTmp.Format("协议： %s/r/n", GetProtocol(g_rawSniffer->m_ipHeader.proto)); strLog += strTmp;
    strTmp.Format("IP源地址： %s/r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.sourceIP)); strLog += strTmp;
    strTmp.Format("IP目标地址: %s/r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.destIP)); strLog += strTmp;
    strTmp.Format("ICMP返回类型：%d/r/n", g_rawSniffer->m_icmpheader.i_type); strLog += strTmp;
    strTmp.Format("ICMP返回代码：%d/r/n", g_rawSniffer->m_icmpheader.i_code); strLog += strTmp;
    strTmp.Format("数据包长度： %d/r/n/r/n/r/n", ntohs(g_rawSniffer->m_ipHeader.total_len)); strLog += strTmp;    
   }
  
   if(g_rawSniffer->m_ipHeader.proto == IPPROTO_UDP && g_rawSniffer->m_bCapUDP)
   {
    nUDPCnt++;
    g_rawSniffer->m_udpheader = *(UDPHEADER*)(recvBuf + iIphLen);
    strTmp.Format("取得 %d UDP包", nUDPCnt); strLog += strTmp;
    strTmp.Format("协议： %s/r/n", GetProtocol(g_rawSniffer->m_ipHeader.proto)); strLog += strTmp;
    strTmp.Format("IP源地址： %s/r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.sourceIP)); strLog += strTmp;
    strTmp.Format("IP目标地址: %s/r/n", inet_ntoa(*(in_addr*)&g_rawSniffer->m_ipHeader.destIP)); strLog += strTmp;
    strTmp.Format("UDP源端口号： %d/r/n", g_rawSniffer->m_udpheader.uh_sport); strLog += strTmp;
    strTmp.Format("UDP目标端口号：%d/r/n", g_rawSniffer->m_udpheader.uh_dport); strLog += strTmp;
    strTmp.Format("数据包长度： %d/r/n", ntohs(g_rawSniffer->m_ipHeader.total_len)); strLog += strTmp;
    strTmp.Format("UDP数据包的报文内容：/r/n"); strLog += strTmp;
    
    ptr = recvBuf + iIphLen + 8;
    cpysize = ntohs(g_rawSniffer->m_ipHeader.total_len) - (iIphLen + 8);
    memcpy(msg, ptr, cpysize);
    
    strTmp.Format("ASCII码格式: /r/n");
    for(int i = 0; i < cpysize; i++)
    {
     if(msg[i] >= 32 && msg[i] < 255)
     {
      strContent.Format("%c",(unsigned char)msg[i]); strLog += strContent;
     }
     else
     {
      strContent.Format("."); strLog += strContent;
     }
    }
    strTmp.Format("/r/n/r/n");  strLog += strTmp;    
    
    strTmp.Format("16进制码格式: /r/n");  strLog += strTmp;
    for(i = 0; i < cpysize; i++)
    {
     strTmp.Format("%2.2X ", (unsigned char)msg[i]);  strLog += strTmp;
    }
    strTmp.Format("/r/n/r/n");  strLog += strTmp;
    
   }
   if(g_rawSniffer->m_CaptureFunc != NULL && strLog.GetLength() > 0 && strContent.GetLength() > 0)
    g_rawSniffer->m_CaptureFunc(strLog);
   Sleep(10);
  }
 }

 // 关闭记录文件
// if(bLogFile)
//  fLog.Close(); 
 return 0;
}
BOOL YRawSniffer::Capture(CaptureDef CaptureFunc /*= NULL*/)
{
 StartAll();
 if(CaptureFunc != NULL)
  m_CaptureFunc = CaptureFunc;
 // 创建线程截取包
 m_bExitCapture = FALSE;
 m_hCaptureThread = CreateThread(NULL, 0, CaptureThread, NULL, 0, NULL);
 if(NULL == m_hCaptureThread)
  TRACE1(" /"m_hCaptureThread = CreateThread(NULL, 0, CaptureThread, NULL, 0, NULL)/" failed! %d ", GetLastError());
 return TRUE;
}
BOOL YRawSniffer::StopCapture()
{
 return ExitAll();
}
```
这个类的使用很简单，声明一个全局的对象（或者成员变量），使用它的Capture和StopCapture就可以了。记得在程序退出的时候也调用一下StopCapture。  
调用Capture 函数的时候，需要把一些过滤条件赋给它：  

```MFC
// 过滤条件
  g_sniffer.m_bCapTCP = m_bCapTCP; // 抓TCP包，TRUE为抓，FALSE不抓
  g_sniffer.m_bCapUDP = m_bCapUDP; // 抓UDP包，TRUE为抓，FALSE不抓
  g_sniffer.m_bCapICMP = m_bCapICMP; // 抓ICMP包，TRUE为抓，FALSE不抓
  g_sniffer.m_strSrcIP = m_strSrcIP; // 包的源IP串，空为不限制
  g_sniffer.m_strDstIP = m_strDstIP; // 包的目标IP串，空为不限制
  g_sniffer.Capture(CapFunc);
```
抓到的包会在一个回调函数中把抓到的包格式化好了传给你。  
例如我的回调函数是这样的：  
```MFC
int CALLBACK CapFunc(CString &strLog)
{
 g_rawDlg->AddLog(strLog); // 把抓到的包在DLG的一个编辑框里输出
 g_log.WriteLogFileRaw(strLog); // 写进一个LOG文件里
 return 0;
}
```
有了上面的这些知识就可以使用这个类实现一个SNIFFER了，很简单的3步：1、把类加进工程 2、包含头文件，声明对象 3、使用这个对象的Capture和StopCapture。  
上面还可以看出我原来的设计是把记录包的功能放在了类里。但是后面觉得这个类还是不要干那么多的活，让记录包的工作都放在上层吧。所以就用了回调函数。  
通过上面代码的研究，你应该很明白原始套接字了吧，这项技术不但可以做SNIFFER，还可以实现洪水攻击等等（因为可以自己构造IP包）。  















