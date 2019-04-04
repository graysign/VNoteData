
Windows应用程序的运行模式是基于消息驱动的，任何线程只要注册了窗口类就会有一个消息队列来接收用户的输入消息和系统消息。为了取得特定线程接收或发送的消息，就要 Windows提供的钩子。
钩子的概念
钩子（Hook）是Windows消息处理机制中的一个监视点，应用程序可以在这里安装一个子程序（钩子函数）以监视指定窗口某种类型的消息，所监视的窗口可以是其他进程创建的。当消息到达后，在目标窗口处理函数处理之前，钩子机制允许应用程序截获它进行处理。
钩子函数是一个处理消息的程序段，通过调用相关的API函数，把它挂入系统。每当特定的消息发出，在没有到达目的窗口前，钩子程序就捕获该消息，亦即钩子函数先得到控制权。这时钩子函数即可以加工处理（改变）该消息，也可以不作处理而继续传递该消息。 
总之，关于Windows钩子要知道以下几点：
1. 钩子是用截获系统中的消息流。利用钩子，可以处理任何感兴趣的消息，包括其他进程的消息。
2. 截获消息后，用于处理消息的子程序叫做钩子函数，它是应用程序自定义的一个函数，在安装钩子时要把这个函数的地址告诉Windows。
3. 系统中同一时间可能有多个进程安装了钩子，多个钩子函数在一起组成钩子链。所以在处理截获到的消息时，应该把消息事件传递下去，以便其他钩子也有机会处理这一消息。
钩子会使得系统变慢，因为它增加了系统对每个消息的处理量。仅应该在必要时才安装钩子，而且在不需要时应尽快移除。
钩子的安装
SetWindowsHookEx函数可以把应用程序定义的钩子函数安装到系统中。
```c++
HHOOK SetWindowsHookEx(
 Int idHook ;       // 指定钩子的类型
 HOOKPROC lpfn;   // 钩子函数的地址。如果使用的是远程钩子，钩子函数必须放在一个DLL中。
 HINSTANCE hMod; // 钩子函数所在DLL的实例句柄。如果是一个局部的钩子，该参数为NULL。
 DWORD    dwThreadID; // 指定要为哪个线程安装钩子。若该值为0被解释成系统范围内的。
）
```
IdHook参数指定了要安装的钩子的类型，可以是下列取值之一：  

WH_CALLWNDPROC       当目标线程调用SendMessage函数发送消息时，钩子函数被调用  .

WH_CALLWNDPROCRET                   当SendMessage发送的消息返回时，钩子函数被调用。  

WH_GETMESSAGE           当目标线程调用GetMessage或者PeekMessage时。 

WH_KEYBOARD               当从消息队列中查询WM_KEYUP或WM_KEYDOWN消息时 

WH_MOUSE                       当调用从消息队列中查询鼠标事件消息时 

WH_MSGFILTER               当对话框，菜单或滚动条要处理一个消息时，钩子函数被调用。该钩子是局部的，它是为哪些有自己消息处理过程的控件对象设计的。 

WH_SYSMSGFILTER        和WH_MSGFILTER一样，只不过是系统范围的。 

WH_JOURNALRECORD 当Windows从硬件队列中获取消息时 。 

WH_JOURNALPLAYBACK       当一个事件从系统的硬件输入队列中别请求时 

WH_SHELL                         当关于Windows外壳事件发生时，比如任务条需要重画它的按钮 

WH_CBT                             当基于计算机的训练（CBT）事件发生时。 

WH_FOREGROUNDIDLE Windows自己使用，一般应用程序很少使用。 

WH_DEBUG                       用来给钩子函数除错。 

Lpfn参数是钩子函数的地址。钩子安装后如果有消息发生，Windows将调用此参数所指向的函数 。
如果dwThreadId参数是0，或者指定一个由其他进程创建的线程ID，lpfn参数指向的钩子函数必须位于一个DLL中。这是因为进程的地址空间是相互隔离的，发生事件的进程不能调用其他进程地址空间的钩子函数。如果钩子函数的实现代码在DLL中，在相关事件发生时，系统会把这个DLL插入到发生事件的进程的地址空间，使它能够调用钩子函数。这种需要将钩子函数写入DLL以便挂钩其他进程事件的钩子称为远程钩子。
如果dwThreadId参数指定一个由自身进程创建的线程ID，lpfn参数指向的钩子函数只要在当前进程中即可，不必非要写入DLL。这种挂钩属于自身进程事件的钩子称为局部钩子。
hMod参数是钩子函数所在DLL的实例句柄，如果钩子函数不再DLL中，应将hMod设置为NULL。
dwThreadId参数指定要与钩子函数相关联的线程ID号。如果设为0，那么钩子就是系统范围内的，即钩子函数将关联到系统内所有线程。
钩子函数
钩子安装后如果有相应的消息发生，Windows将调用SetWindowsHookEx函数指定的钩子函数lpfn。钩子函数的一般形式如下：
```c++
LRESULT CALLBACK HookProc(int nCode, WPARAM wParam, LPARAM lParam)
{
         // 处理该消息的代码 …..
    Return ::CallNextHookEx(hHook,nCode,wParam,lParam);
}
```
HookProc是应用程序的名称。nCode参数是Hook代码，钩子函数使用这个参数来确定任务，它的值依赖于Hook的类型。wParam和lParam参数的值依赖于Hook代码，但是它们典型的值是一些关于发送或者接收消息的信息。
因为系统中可能会有多个钩子的存在，所以要调用那个CallNextHookEx函数把消息传到链中下一个钩子函数。hHook参数是安装钩子时得到的钩子句柄（SetWindowsHookEx的返回值）。
卸载钩子
要卸载钩子，可以调用UnhookWindowsHookEx函数。
```c++
 BOOL UnhookWindowsHookEx(HHOOK hhk); // hhk 为要卸载的钩子的句柄
```
注意：
1．安装钩子的代码可以在DLL模块中，也可以在主模块中，但是一般在DLL里实现它，主要是为了使程序更加模块化。
例子（HOOK键盘消息）
1.dll库的生成（只是部分重要的文件，没有全部贴出）
```c++
//ke
            //The following ifdef block is the standard way of creating macros which make exporting 
// from a DLL simpler. All files within this DLL are compiled with the KEYHOOKLIB_EXPORTS
// symbol defined on the command line. this symbol should not be defined on any project
// that uses this DLL. This way any other project whose source files include this file see 
// KEYHOOKLIB_API functions as being imported from a DLL, wheras this DLL sees symbols
// defined with this macro as being exported.
#ifdef KEYHOOKLIB_EXPORTS
#define KEYHOOKLIB_API __declspec(dllexport)
#else
#define KEYHOOKLIB_API __declspec(dllimport)
#endif

// 自定义与主程序通信的消息
#define HM_KEY WM_USER+1

// 声明要导出的函数
BOOL KEYHOOKLIB_API WINAPI SetKeyHook(BOOL bInstall, 
                               DWORD dwThreadId =0, 
                               HWND hWndCaller = NULL
                               );


// KeyHookLib.cpp : Defines the entry point for the DLL application.
//

#include "stdafx.h"
#include "KeyHookLib.h"

// 共享数据段
#pragma data_seg("YCIShared")
HWND g_hWndCaller = NULL;
HHOOK g_hHook = NULL;
#pragma data_seg()

LRESULT CALLBACK KeyHookProc(int nCode, WPARAM wParam, LPARAM lParam);



// 一个通过内存地址取得模块句柄的帮助函数。
HMODULE    WINAPI ModuleFromAddress(PVOID pv)
{
    MEMORY_BASIC_INFORMATION mbi;
if(VirtualQuery(pv,&mbi,sizeof(mbi)) !=0)
{
return (HMODULE)mbi.AllocationBase;
    }else
{
return NULL;
    }

}


// 安装钩子，卸载钩子的函数
BOOL WINAPI SetKeyHook(
            BOOL bInstall,         // 安装还是卸载已安装的钩子
            DWORD dwThreadId,      // 目标线程的ID
            HWND  hWndCaller)      // 指定主窗口的句柄，钩子函数会向这个窗口发送通知信息。
{
    BOOL bOk;
    g_hWndCaller = hWndCaller;
if(bInstall)
{
        g_hHook = SetWindowsHookEx(
            WH_KEYBOARD,
            KeyHookProc,
            ModuleFromAddress(KeyHookProc),
            dwThreadId);
        bOk = (g_hHook != NULL);
    }
else
{
        bOk = UnhookWindowsHookEx(g_hHook);
        g_hHook = NULL;
    }

return bOk;
}

// 键盘钩子函数
LRESULT CALLBACK KeyHookProc(int nCode, WPARAM wParam, LPARAM lParam)
{
if(nCode<0|| nCode == HC_NOREMOVE )
return CallNextHookEx(g_hHook,nCode,wParam,lParam);
if(lParam &0x40000000) // 消息重复就交给下一个HOOK链
return CallNextHookEx(g_hHook,nCode,wParam,lParam);

// 通知主窗口。wParam参数为虚拟键码，lParam包含了此键的信息
    ::PostMessage(g_hWndCaller,HM_KEY,wParam,lParam);
return CallNextHookEx(g_hHook,nCode,wParam,lParam);
}
// keyhooklib.def
EXPORTS
    SetKeyHook
SECTIONS
    YCIShared    Read Write Shared
```
2.应用
以对话框为基础建立工程（keyhookapp），改动的文件如下：
```c++
// KeyHookAppDlg.cpp : implementation file
//

#include "stdafx.h"
#include "KeyHookApp.h"
#include "KeyHookAppDlg.h"
#include "KeyHookLib.h"
#pragma comment(lib,"KeyHookLib")

#ifdef _DEBUG
#define new DEBUG_NEW
#undef THIS_FILE
staticchar THIS_FILE[] = __FILE__;
#endif

/////////////////////////////////////////////////////////////////////////////
// CAboutDlg dialog used for App About

class CAboutDlg : public CDialog
{
public:
    CAboutDlg();

// Dialog Data
//{{AFX_DATA(CAboutDlg)
enum{ IDD = IDD_ABOUTBOX };
//}}AFX_DATA

// ClassWizard generated virtual function overrides
//{{AFX_VIRTUAL(CAboutDlg)
protected:
virtualvoid DoDataExchange(CDataExchange* pDX);    // DDX/DDV support
//}}AFX_VIRTUAL

// Implementation
protected:
//{{AFX_MSG(CAboutDlg)
//}}AFX_MSG
    DECLARE_MESSAGE_MAP()
};

CAboutDlg::CAboutDlg() : CDialog(CAboutDlg::IDD)
{
//{{AFX_DATA_INIT(CAboutDlg)
//}}AFX_DATA_INIT
}

void CAboutDlg::DoDataExchange(CDataExchange* pDX)
{
    CDialog::DoDataExchange(pDX);
//{{AFX_DATA_MAP(CAboutDlg)
//}}AFX_DATA_MAP
}

BEGIN_MESSAGE_MAP(CAboutDlg, CDialog)
//{{AFX_MSG_MAP(CAboutDlg)
// No message handlers
//}}AFX_MSG_MAP
END_MESSAGE_MAP()

/////////////////////////////////////////////////////////////////////////////
// CKeyHookAppDlg dialog

CKeyHookAppDlg::CKeyHookAppDlg(CWnd* pParent /*=NULL*/)
    : CDialog(CKeyHookAppDlg::IDD, pParent)
{
//{{AFX_DATA_INIT(CKeyHookAppDlg)
// NOTE: the ClassWizard will add member initialization here
//}}AFX_DATA_INIT
// Note that LoadIcon does not require a subsequent DestroyIcon in Win32
    m_hIcon = AfxGetApp()->LoadIcon(IDR_MAINFRAME);
}

void CKeyHookAppDlg::DoDataExchange(CDataExchange* pDX)
{
    CDialog::DoDataExchange(pDX);
//{{AFX_DATA_MAP(CKeyHookAppDlg)
// NOTE: the ClassWizard will add DDX and DDV calls here
//}}AFX_DATA_MAP
}

BEGIN_MESSAGE_MAP(CKeyHookAppDlg, CDialog)
//{{AFX_MSG_MAP(CKeyHookAppDlg)
    ON_WM_SYSCOMMAND()
    ON_WM_PAINT()
    ON_WM_QUERYDRAGICON()
    ON_MESSAGE(HM_KEY,OnHookKey)
//}}AFX_MSG_MAP
END_MESSAGE_MAP()

/////////////////////////////////////////////////////////////////////////////
// CKeyHookAppDlg message handlers

BOOL CKeyHookAppDlg::OnInitDialog()
{
    CDialog::OnInitDialog();

// Add "About..." menu item to system menu.

// IDM_ABOUTBOX must be in the system command range.
    ASSERT((IDM_ABOUTBOX &0xFFF0) == IDM_ABOUTBOX);
    ASSERT(IDM_ABOUTBOX <0xF000);

    CMenu* pSysMenu = GetSystemMenu(FALSE);
if (pSysMenu != NULL)
{
        CString strAboutMenu;
        strAboutMenu.LoadString(IDS_ABOUTBOX);
if (!strAboutMenu.IsEmpty())
{
            pSysMenu->AppendMenu(MF_SEPARATOR);
            pSysMenu->AppendMenu(MF_STRING, IDM_ABOUTBOX, strAboutMenu);
        }
    }

// Set the icon for this dialog.  The framework does this automatically
//  when the application's main window is not a dialog
    SetIcon(m_hIcon, TRUE);            // Set big icon
    SetIcon(m_hIcon, FALSE);        // Set small icon

// TODO: Add extra initialization here

// 安装钩子
if(!SetKeyHook(TRUE,0,m_hWnd))
        MessageBox("安装钩子失败");


return TRUE;  // return TRUE  unless you set the focus to a control
}

void CKeyHookAppDlg::OnSysCommand(UINT nID, LPARAM lParam)
{
if ((nID &0xFFF0) == IDM_ABOUTBOX)
{
        CAboutDlg dlgAbout;
        dlgAbout.DoModal();
    }
else
{
        CDialog::OnSysCommand(nID, lParam);
    }
}

// If you add a minimize button to your dialog, you will need the code below
//  to draw the icon.  For MFC applications using the document/view model,
//  this is automatically done for you by the framework.

void CKeyHookAppDlg::OnPaint() 
{
if (IsIconic())
{
        CPaintDC dc(this); // device context for painting

        SendMessage(WM_ICONERASEBKGND, (WPARAM) dc.GetSafeHdc(), 0);

// Center icon in client rectangle
int cxIcon = GetSystemMetrics(SM_CXICON);
int cyIcon = GetSystemMetrics(SM_CYICON);
        CRect rect;
        GetClientRect(&rect);
int x = (rect.Width() - cxIcon +1) /2;
int y = (rect.Height() - cyIcon +1) /2;

// Draw the icon
        dc.DrawIcon(x, y, m_hIcon);
    }
else
{
        CDialog::OnPaint();
    }
}

// The system calls this to obtain the cursor to display while the user drags
//  the minimized window.
HCURSOR CKeyHookAppDlg::OnQueryDragIcon()
{
return (HCURSOR) m_hIcon;
}

void CKeyHookAppDlg::OnCancel() 
{
// TODO: Add extra cleanup here
// 卸载钩子
    SetKeyHook(FALSE);
    CDialog::OnCancel();
}

void CKeyHookAppDlg::OnOK() 
{
// TODO: Add extra validation here
// 卸载钩子
    SetKeyHook(FALSE);
    CDialog::OnOK();
}

// 钩子消息处理函数
long CKeyHookAppDlg::OnHookKey(WPARAM wParam,LPARAM lParam)
{
// 此时wParam为用户按键的虚拟键码
// lParam包含按键的重复次数，扫描码，前一个按键状态等信息
// 取得按键名称。lParam是键盘消息的第二个参数
char szKey[80];
    ::GetKeyNameText(lParam,szKey,80);
    CString strItem;
    strItem.Format("用户按键：%s ",szKey);

// 添加到编辑框
    CString strEdit;
    GetDlgItem(IDC_KEY)->GetWindowText(strEdit);
    GetDlgItem(IDC_KEY)->SetWindowText(strItem+strEdit);
    ::MessageBeep(MB_OK);
return0;
}
```

3。运行结果
一个对话框，检测当前键盘的状态。若有按键按下，则在对话框的EDIT控件中显示按下键的名字。