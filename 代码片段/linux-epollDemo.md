```c++
#ifndef _TCP_CONTROL_H
#define _TCP_CONTROL_H

#include <stdio.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <netdb.h>
#include <stdlib.h>

#define SERV_PORT   9000 
#define MAX_EVENT   20 




struct UserData
{
    int             nClientFd;
    int             nReadIndex;
    int             nMessageLength;
    unsigned char   szMessageBuffer[255];
};

class TcpControl
{

public:
    TcpControl(/* args */);
    ~TcpControl();

    int     Init();
    void    ReceiveMessage();
private:
    void   _AcceptClient();
    int    _ParseHeader(struct UserData* dao);
    int    _ParseBody(struct UserData* dao);
    void   _DestoryClient(struct  epoll_event epEvent);
private:
    int     m_nEpollFd;
    int     m_nListenFd;
    
    struct  epoll_event m_epEvent[MAX_EVENT];
};

#endif // !_TCP_CONTROL_H
```

```c++
//LOG
#include <stdio.h>
#include <stdarg.h>
#include <time.h>
void  LOG(const char *format, ...)
{
	va_list argp;
	char buf[1024]; 
	 
	va_start(argp, format); 
	vsnprintf(buf, sizeof(buf), format, argp); 
	va_end(argp);

    time_t now;     
    struct tm  *timenow;
    time(&now); 
    timenow = localtime(&now); 
    printf("[DEBUG %2.u:%2.u:%2.u] %s ", timenow->tm_hour, timenow->tm_min, timenow->tm_sec, buf); 
}
```



```c++
#include "TcpControl.h"


TcpControl::TcpControl(/* args */)
{
    m_nEpollFd  =   -1;
    m_nListenFd =   -1;
}

TcpControl::~TcpControl()
{
    if(m_nListenFd > 0)
        close(m_nListenFd);
    if(m_nEpollFd)
        close(m_nEpollFd);
    m_nEpollFd  =   -1;
    m_nListenFd =   -1;
}


int     
TcpControl::Init()
{
    LOG("TcpControl::Init Begin \n");
    m_nListenFd = socket(AF_INET, SOCK_STREAM, 0);
    if (m_nListenFd == -1) {
        LOG("TcpControl::Init Open listen socket error! \n");
        return -1;
    }
    /* Enable address reuse */
    int on = 1;
    // 打开 socket 端口复用, 防止测试的时候出现 Address already in use
    int result = setsockopt( m_nListenFd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on) );
    if (result == -1) { 
        LOG("TcpControl::Init Set listen socket SO_REUSEADDR error! \n");
        return -1;
    }

    struct sockaddr_in servaddr;
    servaddr.sin_family         =   AF_INET;
    servaddr.sin_addr.s_addr    =   htonl(INADDR_ANY);
    servaddr.sin_port           =   htons(SERV_PORT);

    result  =   bind(m_nListenFd, (struct sockaddr*)&servaddr, sizeof(servaddr));
    if (-1 == result) { 
        LOG("TcpControl::Init Set Bind port error! \n");
        return -1;
    }
    result  =   listen(m_nListenFd, 20);
    if (-1 == result) { 
        LOG("TcpControl::Init Start listen error! \n");
        return -1;
    }
    //create epoll
    m_nEpollFd      =   epoll_create1(0);//EPOLLONESHOT
    struct epoll_event  tep;
    tep.events      =   EPOLLIN;
    tep.data.ptr     =   (void*)&m_nListenFd;
    result          =   epoll_ctl(m_nEpollFd, EPOLL_CTL_ADD, m_nListenFd, &tep);
    if(-1 == result) {
        LOG("TcpControl::Init Set epoll_ctl Error! \n");
        return -1;
    }
    LOG("TcpControl::Init Finish \n");
    return 0;
}


void    
TcpControl::ReceiveMessage()
{
    LOG("TcpControl::ReceiveMessage Begin \n");
    if(m_nEpollFd == -1 || m_nListenFd == -1){
        LOG("TcpControl::ReceiveMessage m_nEpollFd[%d] or m_nListenFd[%d] -1 Error! \n",m_nEpollFd, m_nListenFd);
        return;
    }
    
    while(1){
        int wait_count = epoll_wait(m_nEpollFd, m_epEvent, MAX_EVENT, -1);
        for (int i = 0 ; i < wait_count; i++) {
            uint32_t events = m_epEvent[i].events;
            // 判断epoll是否发生错误
            if ( events & EPOLLERR || events & EPOLLHUP || (! events & EPOLLIN)) {
                LOG("TcpControl::ReceiveMessage Epoll has error\n");
                _DestoryClient(m_epEvent[i]);
                continue;
            } 
            if (m_epEvent[i].data.ptr == &m_nListenFd) { 
                _AcceptClient();
                continue;
            }
            // 其余事件为 file describe 可以读取
            struct UserData* dao =   (struct UserData*)m_epEvent[i].data.ptr;
            int result_len       =   dao->nMessageLength  ==   -1 ? _ParseHeader(dao) : _ParseBody(dao);
            if(result_len > 0)
                continue;
            LOG("TcpControl::ReceiveMessage Read Error[%d] Closed connection!\n", result_len);
            _DestoryClient(m_epEvent[i]);
        }
    }
    close (m_nEpollFd);
    m_nEpollFd  =   -1;
    LOG("TcpControl::ReceiveMessage Finish \n");
}

void   
TcpControl::_AcceptClient()
{
    LOG("TcpControl::_AcceptClient Begin \n");
    if(m_nEpollFd == -1 || m_nListenFd == -1){
        LOG("TcpControl::_AcceptClient m_nEpollFd[%d] or m_nListenFd[%d] -1 Error! \n",m_nEpollFd, m_nListenFd);
        return;
    }
    char szHostBuf[NI_MAXHOST];// IP地址缓存
    char szPortBuf[NI_MAXSERV];// PORT缓存
    struct sockaddr in_addr     =       { 0 };
    socklen_t in_addr_len       =       sizeof (in_addr);
    int nClientFd               =       accept(m_nListenFd, &in_addr, &in_addr_len);
    if (-1 == nClientFd) {
        LOG("TcpControl::_AcceptClient Accept error\n");
        return;
    }
    int nResult    =       getnameinfo(&in_addr, sizeof (in_addr),szHostBuf, NI_MAXHOST,szPortBuf, NI_MAXSERV, NI_NUMERICHOST | NI_NUMERICSERV);
    if (!nResult) {
        LOG("TcpControl::_AcceptClient New connection:[%s:%s]\n", szHostBuf, szPortBuf);
    }
    //TODO  init client obj
    struct UserData* dao =   (struct UserData*)malloc(sizeof(struct UserData));
    if(dao == NULL){
        LOG("TcpControl::_AcceptClient malloc is NULL Error! \n");
        return;
    }
    dao->nClientFd       =   nClientFd;
    dao->nReadIndex      =   -1;
    dao->nMessageLength  =   -1;
    //add fd
    struct epoll_event  ev;
    //ev.data.fd          =   nClientFd;
    ev.data.ptr          =    (void*)dao;
    ev.events            =   EPOLLIN;
    nResult = epoll_ctl(m_nEpollFd, EPOLL_CTL_ADD, nClientFd, &ev);
    if (-1 == nResult) {
        free(dao);
        LOG("TcpControl::_AcceptClient Set epoll_ctl Error! \n");
    }
    LOG("TcpControl::_AcceptClient Finish \n");
}

int    
TcpControl::_ParseHeader(struct UserData* dao)
{//read header;
    int     nOccupyByte     =   1;
    char    nMesasgeLength  =   0;
    u_long  nHasReadByte    =   0; 
    ioctl(dao->nClientFd, FIONREAD, &nHasReadByte);
    if(nHasReadByte < nOccupyByte)
        return nHasReadByte;
    LOG("TcpControl::_ParseHeader Message Has Read Byte[%d] \n", nHasReadByte);
    int     nReadLength     =    read(dao->nClientFd,  &nMesasgeLength, nOccupyByte);
    if(nReadLength > 0){
        dao->nMessageLength =    nMesasgeLength - 1;
        dao->nReadIndex     =    0;
        LOG("TcpControl::_ParseHeader Message Header Length[%d] \n", dao->nMessageLength);
    }
    return nReadLength;
}

int    
TcpControl::_ParseBody(struct UserData* dao)
{//read body 
    int     nReadLength     =    read(dao->nClientFd,  (void*)(dao->szMessageBuffer + dao->nReadIndex), dao->nMessageLength - dao->nReadIndex);
    if(nReadLength <= 0)
        return nReadLength;
    
    dao->nReadIndex  =    dao->nReadIndex + nReadLength;
    LOG("TcpControl::_ParseBody Message Body Read[%d] Aready[%d] Need[%d]\n", nReadLength, dao->nReadIndex,dao->nMessageLength - dao->nReadIndex);
    if(dao->nReadIndex != dao->nMessageLength)
        return nReadLength;
        
    //todo  handle message
    write(STDOUT_FILENO, dao->szMessageBuffer, dao->nMessageLength);
    printf("\n") ;
    //set 
    dao->nReadIndex      =   -1;
    dao->nMessageLength  =   -1;
    
    return nReadLength;
}

void   
TcpControl::_DestoryClient(struct  epoll_event epEvent)
{
    LOG("TcpControl::_DestoryClient begin \n");
    int  nClientFd  =   -1;
    if(epEvent.data.ptr == &m_nListenFd){
        nClientFd   =   epEvent.data.fd;
    }else{
        struct UserData* dao =  (struct UserData*)epEvent.data.ptr;
        nClientFd   =   dao->nClientFd;
        free(dao); 
    }
    epoll_ctl(m_nEpollFd, EPOLL_CTL_DEL, nClientFd ,NULL);
    close(nClientFd);
    LOG("TcpControl::_DestoryClient finish \n");
}
```