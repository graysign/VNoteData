&emsp;&emsp; 在之前的博文中，我们主要介绍了如何使用WMI查询信息和接收事件。本文将介绍WMI的另一种用法——执行方法。  
&emsp;&emsp;这块的内容在msdn中有详细的介绍，如果想看原版的可以参阅[《Example: Calling a Provider Method》](https://msdn.microsoft.com/en-us/library/aa390421(v=vs.85).aspx)  
&emsp;&emsp;本文将基于《WMI技术介绍和应用——VC开发WMI应用的基本步骤》中介绍的基类CWMI，在继承类中重写Excute函数，实现执行方法的功能。  
&emsp;&emsp;之所以继承于CWMI，是为了将WMI逻辑中相同部分提炼出来。不同的使用方式只要完成其核心功能即可，回顾下CWMI类的执行主体  
```c++
HRESULT CWMI::ExcuteFun()  
{  
    HRESULT hr = E_FAIL;  
    CComPtr<IWbemLocator> pLoc = NULL;  
    CComPtr<IWbemServices> pSvc = NULL;  
  
    do {  
        hr = InitialCom();  
        CHECKHR(hr);  
  
        hr = SetComSecLevels();  
        CHECKHR(hr);  
  
        hr = ObtainLocator2WMI(pLoc);  
        CHECKHR(hr);  
  
        hr = Connect2WMI(pLoc, pSvc);  
        CHECKHR(hr);  
  
        hr = SetProxySecLevels(pSvc);  
        CHECKHR(hr);  
  
        hr = Excute(pSvc);  
        CHECKHR(hr);  
  
    } while (0);  
    return hr;  
}  
```

&emsp;&emsp;除Excute方法之外的其他方法，我们都是公用的。那我们看下执行方法的类是如何实现的。  
&emsp;&emsp;首先我们定义一个map，用于保存执行函数的参数。因为不同函数的参数名不同，类型不同，所以我们要做如下定义  
```c++
typedef std::map<std::wstring, CComVariant> ParamsMap;  
  
class CExcuteMethod : public CWMI  
{  
public:  
    CExcuteMethod(const wstring& wszNamespace, const std::wstring& wstrClass,  
        const std::wstring& wstrInstanceName, const std::wstring& wstrMethod, const std::wstring& wstrRet, const ParamsMap& params);  
    ~CExcuteMethod(void);  
private:  
    HRESULT Excute(CComPtr<IWbemServices> pSvc);  
private:  
    std::wstring m_wstrInstanceName;  
    std::wstring m_wstrClassName;  
    std::wstring m_wstrMethod;  
    std::wstring m_wstrRet;  
    ParamsMap m_params;  
};  
```

&emsp;&emsp;CComVariant类型保证我们每个参数可以由使用者自己定义。  
&emsp;&emsp;在构造函数中，我们需要传入WMI类名（非C++类名），调用方法名，返回值名，参数map。  
&emsp;&emsp;在执行的主体函数Excute中，我们首先使用WMI类名获取类  
```c++
HRESULT CExcuteMethod::Excute( CComPtr<IWbemServices> pSvc )  
{  
    HRESULT hr = WBEM_S_FALSE;  
  
    do {  
        CComBSTR bstrClassName = m_wstrClassName.c_str();  
        CComPtr<IWbemClassObject> spClassObject = NULL;  
        hr = pSvc->GetObject(bstrClassName, 0, NULL, &spClassObject, NULL);  
        if (FAILED(hr) || !spClassObject) {  
            break;  
        }  
```

&emsp;&emsp;然后通过方法名，获取这个类中函数的入参定义  
```c++
CComBSTR bstrMethodName = m_wstrMethod.c_str();  
CComPtr<IWbemClassObject> spInParamsDefinition = NULL;  
hr = spClassObject->GetMethod(bstrMethodName, 0, &spInParamsDefinition, NULL);  
if (FAILED(hr) || !spInParamsDefinition) {  
    break;  
}  
```

&emsp;&emsp;实例化入参定义的一个实例，然后对这个入参实例进行参数赋值。注意一下，这个地方的赋值，非常符合脚本语言的问题，即通过名称进行赋值，而不是按照C语言风格的入参顺序进行赋值  
```c++
CComPtr<IWbemClassObject> spParamsInstance = NULL;  
hr = spInParamsDefinition->SpawnInstance(0, &spParamsInstance);  
if (FAILED(hr) || !spParamsInstance) {  
    break;  
}  
  
for (ParamsMap::iterator it = m_params.begin(); it != m_params.end(); it++) {  
    if (!it->first.empty()) {  
        CComVariant value = it->second;  
        hr = spParamsInstance->Put(it->first.c_str(), 0, &value, 0);  
    }  
}  
```

&emsp;&emsp; 最后，我们通过类名、方法名、参数实例去执行方法名获取返回的数据实例  
```c++
CComPtr<IWbemClassObject> spOutParams = NULL;  
if (!m_wstrInstanceName.empty()) {  
    //CComBSTR(L"ClassInstance.name='InstanceName'")  
    CComBSTR bstrInstanceName = m_wstrInstanceName.c_str();  
    hr = pSvc->ExecMethod(bstrInstanceName, bstrMethodName, 0, NULL, spParamsInstance, &spOutParams, NULL);  
}  
else if (!m_wstrClassName.empty()) {  
    hr = pSvc->ExecMethod(bstrClassName, bstrMethodName, 0, NULL, spParamsInstance, &spOutParams, NULL);  
}  
  
if (SUCCEEDED(hr) && !m_wstrRet.empty()) {  
    CComVariant varRet;  
    hr = spOutParams->Get(CComBSTR(m_wstrRet.c_str()), 0, &varRet, NULL, 0);  
}  
```

&emsp;&emsp;如此便可以执行一个方法调用。这儿有个地方需要注意下，就是调用ExecMethod方式存在两种方式:  
&emsp;&emsp;1. 类的静态方法直接使用类名调用  
&emsp;&emsp;2. 类的非静态方法使用对象名调用，这种调用我们将在之后的讲解WMI Provider时介绍。  
&emsp;&emsp;我们在main函数中进行如下测试  
```c++
CExcuteMethod* excute_method = NULL;  
{  
    ParamsMap params;  
    {  
        CComVariant vt;  
        vt.vt = VT_BSTR;  
        vt.bstrVal = CComBSTR("notepad.exe");  
        params[L"CommandLine"] = vt;  
    }  
  
    excute_method = new CExcuteMethod(L"root\\CIMV2", L"", L"Win32_Process", L"Create", L"ReturnValue", params);  
    excute_method->ExcuteFun();  
}  
delete excute_method;  
```

&emsp;&emsp;这段代码将启动一个记事本程序，我们没有指定对象名，说明Ceate方法是Win32_Process的静态方法。这段代码需要注意下，就是我为什么要使用动态创建指针的方式，而不是直接定义一个对象？还有就是为什么要种{}控制除指针申明和销毁之外的逻辑？这其中主要的原因是我们CWMI类中控制了COM组件的初始化和卸载操作。如果直接使用对象，则对象的消亡和Main函数中使用的CComVariant类型数据的消亡顺序将不可控制，会导致崩溃（实际的确是CComVariant后释放从而出现异常）。而我们使用动态创建对象和使用{}控制CComVariant的生命周期的方式，实现了对象消亡顺序的可控。当然这不是一种好的设计，于是我们似乎应该将WMI的初始化和卸载交由调用者控制。