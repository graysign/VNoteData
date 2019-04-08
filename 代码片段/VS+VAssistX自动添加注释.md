# VS+VAssistX自动添加注释

在VC6.0里边，C++函数头注释是使用一个宏完成的，VS系列中C#在函数头输入三个反斜杠也会自动生成XML格式的函数头注释。
又懒得在VS2008中写类似于添加函数头的注释，只能依靠一些工具了,今天给大家介绍VAssistX。
大家可以下载VAssistX插件，安装的时候一定要把VS2008关掉。VAssistX在这就不多做介绍了，大家可以百度或者google之。
以下为大家介绍一下怎么添加函数头注释
随便打开一个C++的工程，找到一个方法，右击函数名，然后依次点击“Refacto”–>“Document Method”，这个时候函数头注释是不是已经出来了，很方便吧。
但是这个注释格式是默认的，可能不适合你的项目。可以在VAssistX的选项中更改显示样式，在VS2008中点击 “VAssistX”–>”Visual VAssistX Options”然后选择Suggestions,再点击”Edit VA Snippets”
在打开的窗口中选择Refactor Document Method，在这就可以更改你的显示样式了。
```c++
//************************************     
// 函数名称: $SymbolName$     
// 函数说明：     
// 作 成 者：Mr.M     
// 作成日期：$DATE$     
// 返 回 值: $SymbolType$     
// 参    数: $MethodArg$     
//************************************  
```

VAssistX 函数注释和文件头注释模板
VAssistX->Insert VA Snippet->Eidt VA Snippets->Refactor Document Method
函数模板
```c++
/*************************************************
// Method: $SymbolName$
// Description: 
// Author: K@yee 
// Date: $DATE$
// Returns: $SymbolType$
// Parameter: $MethodArgName$
// History:
*************************************************/
文件头模板
/*************************************************
// Copyright (C), 2012-2013, CS&S. Co., Ltd.
// File name: $FILE_BASE$.$FILE_EXT$
// Author: K@yee 
// Version: 1.0 
// Date: $DATE$
// Description: 
// Others:
// History:
// <author> K@yee 
// <time> $DATE$
// <version> 1.0 
// <desc> build this moudle 
*************************************************/
``` 