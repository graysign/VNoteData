1. 工作者线程倾向于琐碎的处理，与它不同的是，用户界面线程具有自己的界面而且实际上类似于运行其他应用程序。创建线程而不是其他应用程序的好处是线程可与应用程序共享程序空间，这样可以简化线程与应用程序共享数据的功能。
2. 典型情况是用户界面线程用于完成查询和替换等功能，或者是其他不希望占用主应用程序大量处理时间但是需要一个界面的功能或服务，或者用户也可完全不考虑界面，将这种类型的线程用于窗口消息服务器作为一种传递其消息的方式，以避免使自己因占用处理时间过多而陷入困境。
3. 在时间要求严格的应用程序(例如实时应用程序)中，不希望因为工作者线程启动而等待，这时可将工作者线程中的控制逻辑内置到用户界面线程中并提前创建线程。当需要处理事务时，向用户界面线程发送消息，此时用户界面线程已经运行并且在等待指令。  

2种线程都可以用AfxBeginThread创建，注意不同的是第一个参数。  
```c++
CWinThread* AfxBeginThread( AFX_THREADPROC pfnThreadProc, LPVOID pParam, int nPriority = THREAD_PRIORITY_NORMAL, UINT nStackSize = 0, DWORD dwCreateFlags = 0, LPSECURITY_ATTRIBUTES lpSecurityAttrs = NULL );//WORK线程

CWinThread* AfxBeginThread( CRuntimeClass* pThreadClass, int nPriority = THREAD_PRIORITY_NORMAL, UINT nStackSize = 0, DWORD dwCreateFlags = 0, LPSECURITY_ATTRIBUTES lpSecurityAttrs = NULL );//UI线程
```
下面是2种线程的创建步骤:  

* 工作者线程:  
    * 首先创建一个函数UNIT ThreadProc(LPVOID pParam);可以把这个函数看成一个线程，进行需要的操作。然后在需要调用的时候(比如initialize的时候)调用  
    
```c++
AfxBeginThread(ThreadProc,//ThreadProc控制函数地址
&m_Control,//要显示的区域，控件对象地址，比如EDITBOX LISTBOX
0,0,0,NULL);
```
 启动线程即可  
* 用户界面进程:  
    * 这里以建立好的对话框MFC程序为例，首先用户界面进程需要从CWinThread类派生一个新的线程类，就叫它CMultipleThread吧，这个类下需要有窗体成员CMultipleThreadDlg* m_pDlg用于在窗体上显示，要不然怎么叫界面进程呢？还需要重载虚函数virtual int Run()，在Run函数里进行需要的操作。接着就可以在程序创建该线程  
    ```c++
    CMultipleThread* pThread = (CMultipleThread*)AfxBeginThread(RUNTIME_CLASS(CMultipleThread),THREAD_PRIORITY_NORMAL,0,CREATE_SUSPENDED,NULL);
    pThread->SetOwner(this)//设置窗口指针
    pThread->ResumeThread();//恢复线程
    ```