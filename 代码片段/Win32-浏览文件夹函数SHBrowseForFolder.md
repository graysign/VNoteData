关于SHBrowseForFolder函数和简单使用

打开文件目录对话框，我找到的方法就是使用SHBrowseForFolder函数，这个函数的原型是LPITEMIDLIST SHBrowseForFolder(LPBROWSEINFO lpbi)。函数很简单，就一个返回值和一个参数。参数简单罗列如下
```c++
typedef struct _browseinfo {
     HWND hwndOwner;            // 父窗口句柄
     LPCITEMIDLIST pidlRoot;    // 要显示的文件目录对话框的根(Root)
     LPTSTR pszDisplayName;     // 保存被选取的文件夹路径的缓冲区
     LPCTSTR lpszTitle;         // 显示位于对话框左上部的标题
     UINT ulFlags;              // 指定对话框的外观和功能的标志
     BFFCALLBACK lpfn;          // 处理事件的回调函数
     LPARAM lParam;             // 应用程序传给回调函数的参数
     int iImage;                // 文件夹对话框的图片索引
} BROWSEINFO, *PBROWSEINFO, *LPBROWSEINFO
```
一般而言父窗口句柄（hwndOwner）和根（pidlRoot）设置为Null就可以了，pszDisplayName设定一块MAX_PATH大小的缓冲区，跟显示相关的参数就是对话框提示标题（lpszTitle）、对话框样式（ulFlags）、设定对话框的缺省路径的操作（lpfn和lParam）以及对话框任务栏上显示的图标（iImage）。
由于返回值LPITEMIDLIST是一个指向ITEMIDLIST的指针，这个ITEMIDLIST涉及到Windows Shell中关于管理诸如文件、网络上的计算机、控制面板程序、回收站等等对象的知识点，Windows Shell为了识别具体的每一个对象，就使用了ITEMID来唯一识别和区分，而ITEMIDLIST就是一个完整的对象路径。显然这个函数可以用来浏览非文件对象，比如局域网内的电脑等等，在这里这个LPITEMIDLIST返回的对象路径是一个文件夹的路径，Windows提供了一个函数BOOL SHGetPathFromIDList(LPCITEMIDLIST pidl, LPSTR pszPath)来实现从对象路径转化为文件夹路径。
在这里需要注意的是，这个返回值是通过调用IMalloc Interface来分配内存的，函数并不负责释放内存操作，所以我们在使用完这个返回值之后，必须通过IMalloc Interface来释放内存。
下面给出一段最简单的使用代码
```c++
       BROWSEINFO bi;
       char Buffer[MAX_PATH];
       //初始化入口参数bi开始
       bi.hwndOwner = NULL;
       bi.pidlRoot =NULL;//初始化制定的root目录很不容易
       bi.pszDisplayName = Buffer;//此参数如为NULL则不能显示对话框
       bi.lpszTitle = "选择Sis目标文件路径";
       bi.ulFlags = BIF_EDITBOX;//带编辑框的风格
       bi.lpfn = NULL;
       bi.lParam = 0;
       bi.iImage=IDR_MAINFRAME;
       //初始化入口参数bi结束
       LPITEMIDLIST pIDList = SHBrowseForFolder(&bi);//调用显示选择对话框
       if(pIDList)
       {
          SHGetPathFromIDList(pIDList, Buffer);
          //取得文件夹路径到Buffer里
          CString m_cSisDes = Buffer;//将路径保存在一个CString对象里
       }
       UpdateData(FALSE);
 
       // free memory used     
    IMalloc * imalloc = 0;
       if (SUCCEEDED(SHGetMalloc(&imalloc)))
       {
              imalloc->Free (pIDList);
              imalloc->Release();
       }
```
如上代码可以显示一个最简单的带编辑框的其实选中对象为我的电脑的浏览文件夹对话框。
创建一个可以新建文件夹且指定选中初始路径的浏览文件夹对话框

由于我在实际工作中需要的就是一个有新建文件夹功能且指定初始选中路径的浏览文件夹对话框，就把这个需求当做扩展应用吧，由于对话框样式由ulFlags标记确定，而在系统头文件SHLOBJ.h头文件中给出的对话框样式只有如下几种
```c++
// Browsing for directory.
#define BIF_RETURNONLYFSDIRS   0x0001  // For finding a folder to start document searching
#define BIF_DONTGOBELOWDOMAIN  0x0002  // For starting the Find Computer
#define BIF_STATUSTEXT         0x0004
#define BIF_RETURNFSANCESTORS  0x0008
#define BIF_EDITBOX            0x0010
#define BIF_VALIDATE           0x0020   // insist on valid result (or CANCEL)
 
#define BIF_BROWSEFORCOMPUTER  0x1000  // Browsing for Computers.
#define BIF_BROWSEFORPRINTER   0x2000  // Browsing for Printers
#define BIF_BROWSEINCLUDEFILES 0x4000  // Browsing for Everything
```
没有满足我需求的样式，通过csdn查到其实有一个支持新建文件夹功能的样式值0x40，通常网络上给出宏为BIF_NEWDIALOGSTYLE和BIF_USENEWUI，由于不知道在具体哪个头文件中，所以我们可以在代码中自己定义一下这两个宏，具体如下
```c++
#define BIF_NEWDIALOGSTYLE   0x40
#define BIF_USENEWUI (BIF_NEWDIALOGSTYLE|BIF_EDITBOX)
```
这样一来第一个问题解决了，那么如何让对话框有初始选中的文件夹路径呢，我起初想着通过pidlRoot，结果撞了一鼻子灰，原来设定初始选中文件夹路径，是通过那个神奇的回调函数来实现，换句话来说你调用SHBrowseForFolder也就好比你调用了CDialog:: DoModal()函数，具体这个对话框里面的类似初始化，选择等操作的不同实现就通过lpfn这个回调函数来实现了。
下面给出这个简单扩展的代码
```c++
#define BIF_NEWDIALOGSTYLE   0x40
 
int CALLBACK BrowseCallbackProc(HWND hwnd,UINT uMsg,LPARAM lParam,LPARAM lpData)  
{
       if(uMsg == BFFM_INITIALIZED)
       {  
              SendMessage(hwnd, BFFM_SETSELECTION, TRUE, lpData);
       }
       return 0;  
}
 
void CSisAppendMidDlg::OnButtonSisdes()
{
       // TODO: Add your control notification handler code here
       BROWSEINFO bi;
       char Buffer[MAX_PATH];
       //初始化入口参数bi开始
       bi.hwndOwner = NULL;
       bi.pidlRoot =NULL;//初始化制定的root目录很不容易
       bi.pszDisplayName = Buffer;//此参数如为NULL则不能显示对话框
       bi.lpszTitle = "选择Sis目标文件路径";
       bi.ulFlags = BIF_EDITBOX|BIF_NEWDIALOGSTYLE;  
       CFileFind   finder;
       if(finder.FindFile(m_cSisDes)==FALSE)
       {
              bi.lParam =0;
              bi.lpfn = NULL;
       }
       else
       {
              bi.lParam = (long)(m_cSisDes.GetBuffer(m_cSisDes.GetLength()));//初始化路径，形如(_T("c:\\Symbian"));
              bi.lpfn = BrowseCallbackProc;
       }
       finder.Close();
       bi.iImage=IDR_MAINFRAME;
       //初始化入口参数bi结束
       LPITEMIDLIST pIDList = SHBrowseForFolder(&bi);//调用显示选择对话框
       if(pIDList)
       {
          SHGetPathFromIDList(pIDList, Buffer);
          //取得文件夹路径到Buffer里
          m_cSisDes = Buffer;//将路径保存在一个CString对象里
       }
       UpdateData(FALSE);
 
       // free memory used     
    IMalloc * imalloc = 0;
       if ( SUCCEEDED(SHGetMalloc( &imalloc)))
       {
              imalloc->Free (pIDList);
              imalloc->Release();
       }
 
} 
```
