模态窗口一创建后就会显示,就算设置WS_VISIBLE属性或在OnInitDialog加入ShowWindow(SW_HIDE)也没有效果.

下面这种方法可以比较好的解决这一问题:

首先声明两个变量.
```c++
RECT m_nRect;
LONG m_ExStyle;

//在OnInitDialog事件中加入如下代码用来保存原来的窗口位置和扩展风格.

m_ExStyle = GetWindowLong(hwnd ,GWL_EXSTYLE);
GetWindowRect(hwnd ,&m_nRect);

//核心代码,修改窗口的扩展风格和窗口尺寸

LONG uStyle = m_ExStyle & ~WS_EX_APPWINDOW | WS_EX_TOOLWINDOW ;
SetWindowLong(hwnd ,GWL_EXSTYLE ,uStyle);    
MoveWindow(hwnd ,0 ,0 ,0 ,0 ,FALSE);  

//到了这里窗口已经能够隐藏了,恢复的时候应该怎么办呢?

//在想要显示窗口的时候加入如下代码即可:

 SetWindowLong(hWnd ,GWL_EXSTYLE ,m_ExStyle);                                    //恢复原来的窗口扩展风格和窗口位置
 SetWindowPos(hWnd ,NULL ,m_nRect.left,m_nRect.top ,m_nRect.right – m_nRect.left ,m_nRect.bottom – m_nRect.top, SWP_NOZORDER  | SWP_SHOWWINDOW );
```