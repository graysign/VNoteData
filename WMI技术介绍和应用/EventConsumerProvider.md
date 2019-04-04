&emsp;&emsp;在《WMI技术介绍和应用——Event Provider》和《WMI技术介绍和应用——接收事件》中，我们展现了如何处理和事件相关的WMI知识。而《WMI技术介绍和应用——接收事件》一文则主要讲解了如何查询事件，这种查询是在我们进程存在时发生的，一旦我们进程不存在了，这种查询也无法执行。我们将这种行为称为消费事件，我们执行查询的进程叫做“事件临时消费者”。相对应的，WMI还存在“事件永久消费者”，它并不寄生我们的编写的进程中。本文主要讲解“事件永久消费者”的编写方法  
&emsp;&emsp;由于VS2005没有这种模板，所以我便使用事件提供者模板生成了一个工程，并对这个工程进行改造。我们先从mof文件出发  
&emsp;&emsp;由于我们不需要注册事件提供者，所以我们删除instance of __EventProviderRegistration内容。  
&emsp;&emsp;我们先要声明一个事件消费类  
```c++
class TestEventConsumer : __EventConsumer   
{  
    [key] string name;  
    [write] string value;  
}; 
```
&emsp;&emsp;实例化该类  
```c++
instance of TestEventConsumer as $Consumer  
{  
    name = "TestEventConsumer_Name1";  
    value = "TestEventConsumer_Value";  
};  
```
&emsp;&emsp;再注册一个事件消费者提供者实例  
```c++
instance of __EventConsumerProviderRegistration  
{  
    Provider = $EventConsumer;  
    ConsumerClassNames = {"TestEventConsumer"};  
};  
```
&emsp;&emsp;ConsumerClassNames是一个数组，它记录了事件消费者名。  
&emsp;&emsp;然后要实例化一个筛选器  
```c++
instance of __EventFilter as $Filter  
{  
    Name = "TestEventFilter";  
    QueryLanguage = "WQL";  
    Query = "SELECT * FROM __InstanceCreationEvent WITHIN 1 WHERE TargetInstance ISA 'Win32_USBCOntrollerDevice'";  
    EventNamespace = "\\\\.\\root\\CIMV2";  
}; 
```
&emsp;&emsp; 其中定义了筛选器的名字、查询的语言、查询的命令和需要查询的命名空间。如果没有指定EventNamespace，则使用该mof中默认的命名空间。在本例中，我们要监控USB设备的创建，故要监控CIMV2空间。这儿需要注意的是，我们可以Query内部事件或者外部事件。  
&emsp;&emsp; 最后我们将筛选器绑定到消费者上。  
```c++
instance of __FilterToConsumerBinding  
{  
    Consumer = $Consumer;  
    Filter = $Filter;  
};  
```
&emsp;&emsp; 如此，我们便将mof文件构建好了。  
&emsp;&emsp; 我们再回到cpp文件中，我们首先要定义一个事件消费者Sink  
```c++
class CEventConsumerSink:  
    public CComObjectRootEx<CComMultiThreadModel>,  
    public IWbemUnboundObjectSink   
{   
public:  
    CEventConsumerSink() {}  
    ~CEventConsumerSink()  
    {  
    }  
  
    BEGIN_COM_MAP(CEventConsumerSink)   
        COM_INTERFACE_ENTRY(IWbemUnboundObjectSink)   
    END_COM_MAP()   
      
    // IWbemUnboundObjectSink   
public:   
    STDMETHOD(IndicateToConsumer)(IWbemClassObject* pLogicalConsumer, long lNumObjects, IWbemClassObject** apObjects);  
};  
```
&emsp;&emsp; 我们要实现IndicateToConsumer方法，将事件传到到该方法中处理  

```c++
STDMETHODIMP CEventConsumerSink::IndicateToConsumer(  
    IWbemClassObject* pLogicalConsumer, long lNumObjects, IWbemClassObject** apObjects)  
{  
    ObjectLock lock(this);   
    HRESULT hr = WBEM_S_NO_ERROR;   
  
    CComVariant varClass;   
    hr = pLogicalConsumer->Get(CComBSTR("__CLASS"), 0, &varClass, 0, 0);  
  
    if (0 == _wcsicmp(V_BSTR(&varClass), L"TestEventConsumer")) {  
        for (long lIndex = 0; lIndex < lNumObjects; lIndex++) {  
            CComVariant varEventType;  
            hr = apObjects[lIndex]->Get(CComBSTR("TargetInstance"), 0, &varEventType, 0, 0);  
            CComQIPtr<IWbemClassObject> spEvent = V_UNKNOWN(&varEventType);  
  
            CComVariant varEventTypeClass;  
            hr = spEvent->Get(CComBSTR("__CLASS"), 0, &varEventTypeClass, 0, 0);  
            OutputTrace("CEventConsumerSink::IndicateToConsumer");      
        }    
    }  
    else {  
        hr = WBEM_E_NOT_FOUND;  
    }  
    return hr;   
}  
```

&emsp;&emsp;这段逻辑，我们获取事件名称，并检测其是否是我们需要监控的事件。如果是，则遍历事件数组——发来的事件是一批的。  
&emsp;&emsp;然后我们改造主类。事件消费者提供者不需要继承`IWbemInstProviderImpl`，但是要继承`IWbemEventConsumerProvider`。于是我们在模板生成的代码中将其他无关的方法去掉  
```c++
class ATL_NO_VTABLE CEventConsumer :   
                            public CComObjectRootEx<CComMultiThreadModel>,  
                            public CComCoClass<CEventConsumer, &CLSID_EventConsumer>,  
                            public IWbemProviderInit,  
                            public IWbemEventConsumerProvider  
{      
  public:  
    CEventConsumer(){                    
    }  
  
    ~CEventConsumer(){  
    }  
      
    DECLARE_REGISTRY_RESOURCEID(IDR_EVENTCONSUMER)  
  
    DECLARE_NOT_AGGREGATABLE(CEventConsumer)  
  
    BEGIN_COM_MAP(CEventConsumer)  
        COM_INTERFACE_ENTRY(IWbemProviderInit)  
        COM_INTERFACE_ENTRY(IWbemEventConsumerProvider)  
    END_COM_MAP()  
      
    //IWbemProviderInit  
    HRESULT STDMETHODCALLTYPE Initialize(   
                                         __in_opt LPWSTR pszUser,  
                                         LONG lFlags,  
                                         __in LPWSTR pszNamespace,  
                                         __in_opt LPWSTR pszLocale,  
                                         IWbemServices *pNamespace,  
                                         IWbemContext *pCtx,  
                                         IWbemProviderInitSink *pInitSink);  
  
    // IWbemEventConsumerProvider  
    HRESULT STDMETHODCALLTYPE FindConsumer(   
            /* [in] */ IWbemClassObject *pLogicalConsumer,  
            /* [out] */ IWbemUnboundObjectSink **ppConsumer);  
  
};  
```

&emsp;&emsp; Initialize中删除模板生成的多余代码就可以了，我们主要要实现的就是FindConsumer方法  
```c++
STDMETHODIMP CEventConsumer::FindConsumer(   
            /* [in] */ IWbemClassObject *pLogicalConsumer,  
            /* [out] */ IWbemUnboundObjectSink **ppConsumer)  
{  
    HRESULT hr = WBEM_E_NOT_FOUND;  
    *ppConsumer = NULL;  
  
    CComVariant varClass;  
    hr = pLogicalConsumer->Get(L"__Class", 0, &varClass, 0, 0);  
  
    if (0 == _wcsicmp(V_BSTR(&varClass), L"TestEventConsumer")) {  
        CComObject<CEventConsumerSink>* pEventSink = NULL;  
        hr = CComObject<CEventConsumerSink>::CreateInstance(&pEventSink);  
        if (FAILED(hr)) {  
            return hr;  
        }  
        hr = pEventSink->QueryInterface(IID_IWbemUnboundObjectSink, (LPVOID*)ppConsumer);  
        if (FAILED(hr)) {  
            return hr;  
        }  
    }  
    else {  
        hr = WBEM_E_NOT_FOUND;  
    }  
    return hr;  
}  
```
&emsp;&emsp; 这段逻辑我们在检测完消费者类是否正确后，就去获取消费者指针。  
&emsp;&emsp;如此Event Consumer Provider便搭建起来了。