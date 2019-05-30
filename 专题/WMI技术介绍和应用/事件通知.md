![](_v_images/_1521450765_29733.png)  

&emsp;&emsp;我们之前介绍的使用WMI查询系统、硬件等信息的功能，是通过查询WMI静态数据的空间实现的。这个功能的核心是在上图中2，即WMI Infrastructure层实现的。本文将让我们对WMI的认识深入到1，即WMI Providers and Managed Objects层。具体的功能是使用WMI检测到WMI数据和服务的变化。再具体一点就是，我们可以使用**WMI检测进程创建、服务状态变化、电脑状态变化，磁盘可用空间变化**等信息。这些都是非常让人激动的技术，我想做过安全的朋友应该清楚，如果想全局监控系统中进程的创建，除了下驱动就是使用Hook技术。如果你只是想知道哪些进程创建这样轻量级的需求，也要使用驱动、Hook这种重量级的技术，是否感觉有点得不偿失？而WMI就给我们提供了这样的一种轻量级的解决方案。  

&emsp;&emsp;上图的WMI Providers and Managed Objects(托管对象和WMI提供者)层中，我们将使用Provider帮我们检测WMI变化。需要注意的一点是，并不是所有的Provider都可以为我们提供事件通知——只有WMI Event Class的托管对象才会在事件发生时给我们提供通知。我们收到的事件，可能来源于两种事件，一种是intrinsic event，即内在事件；还有一种是extrinsic event，即外来事件。内在事件是在标准的WMI数据模型发生改变而产生的事件，这将是我们介绍的重点。外来事件，和内在事件相对，即非标准WMI数据数据模型发生改变而产生的事件。  

&emsp;&emsp;介绍了这么多基础知识了，那如何查询事件通知呢？在《WMI技术介绍和应用——使用VC编写一个半同步查询WMI服务的类》中，我们讲解WMI查询静态数据时，我们可以使用同步查询和半同步查询两种查询方式。像静态数据，正如其名，它是静态的，即它存在就存在，不存在即不存在，所以我们可以使用同步方式查询。**半同步其实就是一个伪装的异步操作**，我们在那篇文章中已经做了介绍，本文不再赘述。而本文主要讲解的查询事件通知，它是动态发生的。即可能我查询的即刻，那个事件还未发生，我们需要等待一段时间，才会在事件发生后接收到通知。**这就意味着查询事件通知，是不可能使用同步查询方式，我们可以选择异步查询或者半同步查询方式。**  

&emsp;&emsp; 作为查询的载体——事件使用者（Event Consumers），也是分为两种：临时事件使用者和永久事件使用者。  

&emsp;&emsp;临时事件使用者是我们未来最早接触到的一个使用者，顾名思义，它是指WMI接收事件通知的生命周期和发起查询的应用程序一致。WMI包含一个统一的接口用来向客户端应用程序提供WMI事件。   

&emsp;&emsp;永久事件使用者是一种更复杂的使用者——它是一个COM对象，用于持续接收WMI事件通知。它使用一些现有的对象和过滤器去获取WMI事件。我们可以设置一些WMI对象和过滤器去获取WMI事件。当一个事件发生，并命中过滤器，WMI将加载永久事件使用者并通知它某事件发生了。或许你会有点好奇，永久事件使用者是保存在什么地方？WMI又是如何找到它的？永久事件使用者是保存在WMI仓库中（上图2层中WMI repository），并且是一个在WMI中注册的可执行文件，这样WMI便可以方便的寻找和加载它了。  

&emsp;&emsp;这些事件都是由事件提供者（An event provider）发送给WMI的。它也是个COM组件。我们可以使用C++或者C#编写事件提供者程序。大部分事件提供者管理着一个WMI对象。对于如何编写WMI事件提供者，我们会在之后介绍。  

&emsp;&emsp;我们再回到查询事件通知，首先我们要编写一个异步事件查询类。因为连接空间等操作和之前的都相同，所以我们的查询类也是继承于《WMI技术介绍和应用——VC开发WMI应用的基本步骤》介绍的CWMI类  
```c++
template<typename T>  
class CAsynNotifyQuery : public CWMI  
{  
public:  
    CAsynNotifyQuery(const wstring& wszNamespace, const wstring& wszWQLQuery, HANDLE hExitEvent);  
    ~CAsynNotifyQuery(void);  
private:  
    HRESULT Excute(CComPtr<IWbemServices> pSvc);  
private:  
     wstring m_wszWQLQuery;  
     HANDLE m_hExitHandle;  
};  
```

&emsp;&emsp; CAsynNotifyQuery是一个模板类，这是区别于我们之前的其他类。我们再来关注下主要方法——Excute方法的实现  
```c++
template<typename T>  
HRESULT CAsynNotifyQuery<T>::Excute( CComPtr<IWbemServices> pSvc )  
{  
    HRESULT hr = WBEM_S_FALSE;  
  
    do {  
        CComPtr<IUnsecuredApartment> pUnsecApp = NULL;  
        hr = CoCreateInstance( CLSID_UnsecuredApartment, NULL,  
            CLSCTX_LOCAL_SERVER, IID_IUnsecuredApartment, (void**) &pUnsecApp);  
        CHECKWMIHR(hr);  
  
        CComPtr<IWbemObjectSink> pSink = new T;  
  
        CComPtr<IUnknown> pStubUnk = NULL;  
        pUnsecApp->CreateObjectStub(pSink, &pStubUnk);  
  
        CComPtr<IWbemObjectSink> pStubSink = NULL;  
        pStubUnk->QueryInterface(IID_IWbemObjectSink, (void**)&pStubSink);  
  
        hr = pSvc->ExecNotificationQueryAsync(   
            CComBSTR("WQL"),  
            CComBSTR(m_wszWQLQuery.c_str()),  
            WBEM_FLAG_SEND_STATUS,  
            NULL,  
            pStubSink );  
  
        CHECKWMIHR(hr);  
  
        if ( NULL != m_hExitHandle ) {  
            WaitForSingleObject(m_hExitHandle, INFINITE );  
        }  
  
        hr = pSvc->CancelAsyncCall(pStubSink);  
  
        if ( NULL != pSink ) {  
            delete pSink;  
            pSink = NULL;  
        }  
  
    } while (0);  
  
    return hr;    
}  
```
&emsp;&emsp; 首先我们需要创建一个IUnsecuredApartment接口实例。该接口是客户进程发起异步调用的，它提供了一个CreateObjectStub方法创建一个桩，WMI将在异步执行过程中对该桩进行操作。我们再看下传入的模板类的定义  

```c++
class CSink : public IWbemObjectSink  
{  
public:  
    CSink();  
    ~CSink();  
  
    virtual ULONG STDMETHODCALLTYPE AddRef();  
    virtual ULONG STDMETHODCALLTYPE Release();  
    virtual HRESULT STDMETHODCALLTYPE QueryInterface(REFIID riid, void** ppv);  
    virtual HRESULT STDMETHODCALLTYPE Indicate(LONG lObjectCount, IWbemClassObject __RPC_FAR* __RPC_FAR* apObjArray);  
    virtual HRESULT STDMETHODCALLTYPE SetStatus(LONG lFlags, HRESULT hResult, BSTR strParam, IWbemClassObject __RPC_FAR* pObjParam);  
    HRESULT DealIWbemClassObject(CComPtr<IWbemClassObject> pObj);  
private:  
    // 返回值为WBEM_S_NO_ERROR则继续枚举，否则中断枚举  
    virtual HRESULT DealWithSingleItem(CComBSTR bstrName, CComVariant Value, CIMTYPE type, LONG lFlavor);  
private:  
    LONG m_lRef;  
    bool m_bDone;  
};  
```

&emsp;&emsp; IWbemObjectSink接口用于接收WMI事件。我们主要需要实现Indicate方法，WMI框架将调用这个方法把消息实例传递给我们。从这个函数的最后一个参数可以看出，它传递过来的是一个事件数组  
```c++
HRESULT STDMETHODCALLTYPE CSink::Indicate( LONG lObjectCount, IWbemClassObject __RPC_FAR* __RPC_FAR* apObjArray )  
{  
    for (long i = 0; i < lObjectCount; i++) {   
        CComPtr<IWbemClassObject> pObj = apObjArray[i];   
        DealIWbemClassObject(pObj);  
    }   
  
    return WBEM_NO_ERROR;   
}  
```

&emsp;&emsp;  DealWbemClassObject是我自己定义和实现的方法，其主要功能就是输出事件的内容。我们可以使用下例去调用查询操作  
```c++
HANDLE hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);  
CAsynNotifyQuery<CInstanceEvent> recvnotify(L"root\\CIMV2", L"SELECT * FROM __InstanceModificationEvent WITHIN 1 WHERE TargetInstance ISA 'Win32_Process'", hEvent);  
recvnotify.ExcuteFun();  
```