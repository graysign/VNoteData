读取和设置xml配置文件是最常用的操作，试用了几个C++的XML解析器，个人感觉TinyXML是使用起来最舒服的，因为它的API接口和Java的十分类似，面向对象性很好。  

TinyXML是一个开源的解析XML的解析库，能够用于C++，能够在Windows或Linux中编译。这个解析库的模型通过解析XML文件，然后在内存中生成DOM模型，从而让我们很方便的遍历这棵XML树。  

DOM模型即文档对象模型，是将整个文档分成多个元素（如书、章、节、段等），并利用树型结构表示这些元素之间的顺序关系以及嵌套包含关系。  

如下是一个XML片段：  

```xml
<Persons>
    <Person ID="1">
        <name>周星星</name>
        <age>20</age>
    </Person>
    <Person ID="2">
        <name>白晶晶</name>
        <age>18</age>
    </Person>
</Persons>

```  

在TinyXML中，根据XML的各种元素来定义了一些类：  

```document
TiXmlBase：整个TinyXML模型的基类。

TiXmlAttribute：对应于XML中的元素的属性。

TiXmlNode：对应于DOM结构中的节点。

TiXmlComment：对应于XML中的注释

TiXmlDeclaration：对应于XML中的申明部分，即<？versiong="1.0" ?>。

TiXmlDocument：对应于XML的整个文档。

TiXmlElement：对应于XML的元素。

TiXmlText：对应于XML的文字部分

TiXmlUnknown：对应于XML的未知部分。 

TiXmlHandler：定义了针对XML的一些操作。
```

TinyXML是个解析库，主要由DOM模型类（TiXmlBase、TiXmlNode、TiXmlAttribute、TiXmlComment、TiXmlDeclaration、TiXmlElement、TiXmlText、TiXmlUnknown）和操作类（TiXmlHandler）构成。它由两个头文件（.h文件）和四个CPP文件（.cpp文件）构成，用的时候，只要将（tinyxml.h、tinystr.h、tinystr.cpp、tinyxml.cpp、tinyxmlerror.cpp、tinyxmlparser.cpp）导入工程就可以用它的东西了。如果需要，可以将它做成自己的DLL来调用。举个例子就可以说明一切。。。  

对应的XML文件：  

```xml
<Persons>
    <Person ID="1">
        <name>phinecos</name>
        <age>22</age>
    </Person>
</Persons>

```

读写XML文件的程序代码：  

```vc++

#include <iostream>
#include "tinyxml.h"
#include "tinystr.h"
#include <string>
#include <windows.h>
#include <atlstr.h>
using namespace std;
 
CString GetAppPath()
{//获取应用程序根目录
    TCHAR modulePath[MAX_PATH];
    GetModuleFileName(NULL, modulePath, MAX_PATH);
    CString strModulePath(modulePath);
    strModulePath = strModulePath.Left(strModulePath.ReverseFind(_T('\\')));
    return strModulePath;
}
 
bool CreateXmlFile(string& szFileName)
{//创建xml文件,szFilePath为文件保存的路径,若创建成功返回true,否则false
    try
    {
        //创建一个XML的文档对象。
        TiXmlDocument *myDocument = new TiXmlDocument();
        //创建一个根元素并连接。
        TiXmlElement *RootElement = new TiXmlElement("Persons");
        myDocument->LinkEndChild(RootElement);
        //创建一个Person元素并连接。
        TiXmlElement *PersonElement = new TiXmlElement("Person");
        RootElement->LinkEndChild(PersonElement);
        //设置Person元素的属性。
        PersonElement->SetAttribute("ID", "1");
        //创建name元素、age元素并连接。
        TiXmlElement *NameElement = new TiXmlElement("name");
        TiXmlElement *AgeElement = new TiXmlElement("age");
        PersonElement->LinkEndChild(NameElement);
        PersonElement->LinkEndChild(AgeElement);
        //设置name元素和age元素的内容并连接。
        TiXmlText *NameContent = new TiXmlText("周星星");
        TiXmlText *AgeContent = new TiXmlText("22");
        NameElement->LinkEndChild(NameContent);
        AgeElement->LinkEndChild(AgeContent);
        CString appPath = GetAppPath();
        string seperator = "\\";
        string fullPath = appPath.GetBuffer(0) +seperator+szFileName;
        myDocument->SaveFile(fullPath.c_str());//保存到文件
    }
    catch (string& e)
    {
        return false;
    }
    return true;
}
 
bool ReadXmlFile(string& szFileName)
{//读取Xml文件，并遍历
    try
    {
        CString appPath = GetAppPath();
        string seperator = "\\";
        string fullPath = appPath.GetBuffer(0) +seperator+szFileName;
        //创建一个XML的文档对象。
        TiXmlDocument *myDocument = new TiXmlDocument(fullPath.c_str());
        myDocument->LoadFile();
        //获得根元素，即Persons。
        TiXmlElement *RootElement = myDocument->RootElement();
        //输出根元素名称，即输出Persons。
        cout << RootElement->Value() << endl;
        //获得第一个Person节点。
        TiXmlElement *FirstPerson = RootElement->FirstChildElement();
        //获得第一个Person的name节点和age节点和ID属性。
        TiXmlElement *NameElement = FirstPerson->FirstChildElement();
        TiXmlElement *AgeElement = NameElement->NextSiblingElement();
        TiXmlAttribute *IDAttribute = FirstPerson->FirstAttribute();
        //输出第一个Person的name内容，即周星星；age内容，即；ID属性，即。
        cout << NameElement->FirstChild()->Value() << endl;
        cout << AgeElement->FirstChild()->Value() << endl;
        cout << IDAttribute->Value()<< endl;
    }
    catch (string& e)
    {
        return false;
    }
    return true;
}
int main()
{
    string fileName = "info.xml";
    CreateXmlFile(fileName);
    ReadXmlFile(fileName);
}

```