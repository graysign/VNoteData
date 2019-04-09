# C#实现网络封包监视
Windows Sockets的一些关于用C#实现的原始套接字(Raw Socket)的编程，以及在此基础上实现的网络封包监视技术。同Winsock1相比，Winsock2最明显的就是支持了Raw Socket  
套接字类型，使用Raw Socket，可把网卡设置成混杂模式，在这种模式下，我们可以收到网络上的IP包，当然包括目的不是本机的IP包，通过原始套接字，我们也可以更加自如地控制  
Windows下的多种协议，而且能够对网络底层的传输机制进行控制。  
在本文例子中，在nbyte.BasicClass命名空间实现了RawSocket类，它包含了我们实现数据包监视的核心技术。在实现这个类之前，需要先写一个IP头结构，  
来暂时存放一些有关网络封包的信息：  
```c#
[StructLayout(LayoutKind.Explicit)]public struct IPHeader 
{
    [FieldOffset(0)] public byte ip_verlen;//I4位首部长度+4位IP版本号　
    [FieldOffset(1)] public byte ip_tos; //8位服务类型TOS　
    [FieldOffset(2)] public ushort ip_totallength; //16位数据包总长度（字节）　
    [FieldOffset(4)] public ushort ip_id; //16位标识　
    [FieldOffset(6)] public ushort ip_offset; //3位标志位　
    [FieldOffset(8)] public byte ip_ttl; //8位生存时间 TTL　
    [FieldOffset(9)] public byte ip_protocol; //8位协议(TCP, UDP, ICMP, Etc.)　
    [FieldOffset(10)] public ushort ip_checksum; //16位IP首部校验和　
    [FieldOffset(12)] public uint ip_srcaddr; //32位源IP地址　
    [FieldOffset(16)] public uint ip_destaddr; //32位目的IP地址
}
```
这样，当每一个封包到达时候，可以用强制类型转化把包中的数据流转化为一个个IPHeader对象。  
下面就开始写RawSocket类了，一开始，先定义几个参数，包括：
```c#
private bool error_occurred;//套接字在接收包时是否产生错误
public bool KeepRunning; //是否继续进行
private static int len_receive_buf; //得到的数据流的长度
byte [] receive_buf_bytes; //收到的字节
private Socket socket = null; //声明套接字
```
还有一个常量：
```c#
const int SIO_RCVALL = unchecked((int)0x98000001);//监听所有的数据包
```
这里的`SIO_RCVAL`L是指示`RawSocket`接收所有的数据包，在以后的`IOContrl`函数中要用，在下面的构造函数中，实现了对一些变量参数的初始化：  
```c#
public RawSocket() //构造函数
{　
    error_occurred=false;　
    len_receive_buf = 4096;　
    receive_buf_bytes = new byte[len_receive_buf];
}
```
下面的函数实现了创建`RawSocket`，并把它与终结点（IPEndPoint：本机IP和端口）绑定：  
```c#
public void CreateAndBindSocket(string IP) //建立并绑定套接字
{　
    socket = new Socket(AddressFamily.InterNetwork, SocketType.Raw, ProtocolType.IP);　
    socket.Blocking = false; //置socket非阻塞状态　
    socket.Bind(new IPEndPoint(IPAddress.Parse(IP), 0)); //绑定套接字　
    if (SetSocketOption()==false) error_occurred=true;
}
```
其中，在创建套接字的一句`socket = new Socket(AddressFamily.InterNetwork, SocketType.Raw, ProtocolType.IP);`中有3个参数：  
第一个参数是设定地址族，MSDN上的描述是“指定 Socket 实例用来解析地址的寻址方案”，当要把套接字绑定到终结点（IPEndPoint）时，需要使用InterNetwork成员，  
即采用IP版本4的地址格式，这也是当今大多数套接字编程所采用一个寻址方案（AddressFamily）。
第二个参数设置的套接字类型就是我们使用的Raw类型了，SocketType是一个枚举数据类型，Raw套接字类型支持对基础传输协议的访问。通过使用 SocketType.Raw，  
你不光可以使用传输控制协议(Tcp)和用户数据报协议(Udp)进行通信，也可以使用网际消息控制协议 (Icmp) 和 Internet 组管理协议 (Igmp) 来进行通信。在发送时，  
您的应用程序必须提供完整的 IP 标头。所接收的数据报在返回时会保持其 IP 标头和选项不变。  
第三个参数设置协议类型，Socket 类使用 ProtocolType 枚举数据类型向 Windows Socket API 通知所请求的协议。这里使用的是IP协议，所以要采用ProtocolType.IP参数。  
在CreateAndBindSocket函数中有一个自定义的SetSocketOption函数，它和Socket类中的SetSocketOption不同，我们在这里定义的是具有IO控制功能的SetSocketOption，  
它的定义如下：  
```c#
private bool SetSocketOption() //设置raw socket
{　
    bool ret_value = true;　
    try　{　　
        socket.SetSocketOption(SocketOptionLevel.IP, SocketOptionName.HeaderIncluded, 1);　　
        byte []IN = new byte[4]{1, 0, 0, 0};　　
        byte []OUT = new byte[4];　　//低级别操作模式,接受所有的数据包，这一步是关键，必须把socket设成raw和IP Level才可用　　SIO_RCVALL　　
        int ret_code = socket.IOControl(SIO_RCVALL, IN, OUT);　　
        ret_code = OUT[0] + OUT[1] + OUT[2] + OUT[3];//把4个8位字节合成一个32位整数　　
        if(ret_code != 0) 
            ret_value = false;　
    }　catch(SocketException)　{　　
        ret_value = false;　
    }　
    return ret_value;
}
```
其中，设置套接字选项时必须使套接字包含IP包头，否则无法填充IPHeader结构，也无法获得数据包信息。  `int ret_code = socket.IOControl(SIO_RCVALL, IN, OUT);`  
是函数中最关键的一步了，因为，在windows中我们不能用Receive函数来接收raw socket上的数据，这是因为，所有的IP包都是先递交给系统核心，然后再传输到用户程序，当发送  
一个raws socket包的时候（比如syn），核心并不知道，也没有这个数据被发送或者连接建立的记录，因此，当远端主机回应的时候，系统核心就把这些包都全部丢掉，从而到不了应  
用程序上。所以，就不能简单地使用接收函数来接收这些数据报。要达到接收数据的目的，就必须采用嗅探，接收所有通过的数据包，然后进行筛选，留下符合我们需要的。可以通过  
设置SIO_RCVALL，表示接收所有网络上的数据包。接下来介绍一下IOControl函数。MSDN解释它说是设置套接字为低级别操作模式，怎么低级别操作法？其实这个函数与API中的  
WSAIoctl函数很相似。WSAIoctl函数定义如下：  
```c#
int WSAIoctl(　
    SOCKET s, //一个指定的套接字　
    DWORD dwIoControlCode, //控制操作码　
    LPVOID lpvInBuffer, //指向输入数据流的指针　
    DWORD cbInBuffer, //输入数据流的大小（字节数）　
    LPVOID lpvOutBuffer, // 指向输出数据流的指针　
    DWORD cbOutBuffer, //输出数据流的大小（字节数）　
    LPDWORD lpcbBytesReturned, //指向输出字节流数目的实数值　
    LPWSAOVERLAPPED lpOverlapped, //指向一个WSAOVERLAPPED结构　
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine//指向操作完成时执行的例程
);
```
C#的IOControl函数不像WSAIoctl函数那么复杂，其中只包括其中的控制操作码、输入字节流、输出字节流三个参数，不过这三个参数已经足够了。我们看到函数中定义了一个字节数  
组：`byte []IN = new byte[4]{1, 0, 0, 0`}实际上它是一个值为1的DWORD或是Int32，同样`byte []OUT = new byte[4];`也是，它整和了一个int,作为WSAIoctl函数中参  
数lpcbBytesReturned指向的值。  
    因为设置套接字选项时可能会发生错误，需要用一个值传递错误标志：  
```c#
public bool ErrorOccurred{　
    get　{　
        return error_occurred;　
    }
}
```
下面的函数实现的数据包的接收：  
```c#
//解析接收的数据包，形成PacketArrivedEventArgs事件数据类对象，并引发PacketArrival事件unsafe 
private void Receive(byte [] buf, int len){　
    byte temp_protocol=0;　
    uint temp_version=0;　
    uint temp_ip_srcaddr=0;　
    uint temp_ip_destaddr=0;　
    short temp_srcport=0;　
    short temp_dstport=0;　
    IPAddress temp_ip;　
    PacketArrivedEventArgs e=new PacketArrivedEventArgs();//新网络数据包信息事件　
    fixed(byte *fixed_buf = buf)　{　　
        IPHeader * head = (IPHeader *) fixed_buf;//把数据流整和为IPHeader结构　　
        e.HeaderLength=(uint)(head->ip_verlen & 0x0F) << 2;　　
        temp_protocol = head->ip_protocol;　　
        switch(temp_protocol)//提取协议类型　　
        {　　　
            case 1: e.Protocol="ICMP"; break;　　　
            case 2: e.Protocol="IGMP"; break;　　　
            case 6: e.Protocol="TCP"; break;　　　
            case 17: e.Protocol="UDP"; break;　　　
            default: e.Protocol= "UNKNOWN"; break;　　
        }　　
        temp_version =(uint)(head->ip_verlen & 0xF0) >> 4;//提取IP协议版本　　
        e.IPVersion = temp_version.ToString();　　//以下语句提取出了PacketArrivedEventArgs对象中的其他参数　　
        temp_ip_srcaddr = head->ip_srcaddr;　　
        temp_ip_destaddr = head->ip_destaddr;　　
        temp_ip = new IPAddress(temp_ip_srcaddr);　　
        e.OriginationAddress =temp_ip.ToString();　　
        temp_ip = new IPAddress(temp_ip_destaddr);　　
        e.DestinationAddress = temp_ip.ToString();　　
        temp_srcport = *(short *)&fixed_buf[e.HeaderLength];　　
        temp_dstport = *(short *)&fixed_buf[e.HeaderLength+2];　　
        e.OriginationPort=IPAddress.NetworkToHostOrder(temp_srcport).ToString();　　
        e.DestinationPort=IPAddress.NetworkToHostOrder(temp_dstport).ToString();　　
        e.PacketLength =(uint)len;　　
        e.MessageLength =(uint)len - e.HeaderLength;　　
        e.ReceiveBuffer=buf;　　//把buf中的IP头赋给PacketArrivedEventArgs中的IPHeaderBuffer　　
        Array.Copy(buf,0,e.IPHeaderBuffer,0,(int)e.HeaderLength);　　//把buf中的包中内容赋给PacketArrivedEventArgs中的MessageBuffer　　
        Array.Copy(buf,(int)e.HeaderLength,e.MessageBuffer,0,(int)e.MessageLength);　
    }　
    //引发PacketArrival事件　
    OnPacketArrival(e);
}
```
大家注意到了，在上面的函数中，我们使用了指针这种所谓的不安全代码，可见在C#中指针和移位运算这些原始操作也可以给程序员带来编程上的便利。在函数中声明  
PacketArrivedEventArgs类对象，以便通过OnPacketArrival(e)函数通过事件把数据包信息传递出去。其中PacketArrivedEventArgs类是RawSocket类中的嵌套类，它继承了系统事件  
（Event）类，封装了数据包的IP、端口、协议等其他数据包头中包含的信息。在启动接收数据包的函数中，我们使用了异步操作的方法，以下函数开启了异步监听的接口：  
```c#
public void Run() //开始监听
{　
    IAsyncResult ar = socket.BeginReceive(receive_buf_bytes, 0, len_receive_buf, SocketFlags.None, new AsyncCallback(CallReceive), this);
}
```
Socket.BeginReceive函数返回了一个异步操作的接口，并在此接口的生成函数BeginReceive中声明了异步回调函数CallReceive，并把接收到的网络数据流传给receive_buf_bytes，  
这样就可用一个带有异步操作的接口参数的异步回调函数不断地接收数据包：  
```c#
private void CallReceive(IAsyncResult ar)//异步回调
{　
    int received_bytes;　
    received_bytes = socket.EndReceive(ar);　
    Receive(receive_buf_bytes, received_bytes);　
    if (KeepRunning) 
        Run();
}
```
此函数当挂起或结束异步读取后去接收一个新的数据包，这样能保证让每一个数据包都能够被程序探测到。  
下面通过声明代理事件句柄来实现和外界的通信：  
```c#
public delegate void PacketArrivedEventHandler(Object sender, PacketArrivedEventArgs args);//事件句柄：包到达时引发事件
public event PacketArrivedEventHandler PacketArrival;//声明时间句柄函数
```
这样就可以实现对数据包信息的获取，采用异步回调函数，可以提高接收数据包的效率，并通过代理事件把封包信息传递到外界。既然能把所有的封包信息传递出去，就可以实现对  
数据包的分析了：）不过RawSocket的任务还没有完，最后不要望了关闭套接字啊：  
```c#
public void Shutdown() //关闭raw socket
{　
    if(socket != null)　{　　
        socket.Shutdown(SocketShutdown.Both);　　
        socket.Close();　
    }
}
```
以上介绍了RawSocket类通过构造IP头获取了包中的信息，并通过异步回调函数实现了数据包的接收，并使用时间代理句柄和自定义的数据包信息事件类把数据包信息发送出去，从而  
实现了网络数据包的监视，这样我们就可以在外部添加一些函数对数据包进行分析了。  


































　