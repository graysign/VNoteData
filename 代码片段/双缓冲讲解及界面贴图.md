# 双缓冲讲解及界面贴图
1. 原理：双缓冲的原理可以这样形象的理解：把电脑屏幕看作一块黑板。首先我们在内存环境中建立一个“虚拟“的黑板，
然后在这块黑板上绘制复杂的图形，等图形全部绘制完毕的时候，再一次性的把内存中绘制好的图形“拷贝”到另一块黑板（屏幕）上。
采取这种方法可以提高绘图速度，极大的改善绘图效果。
 
2. 具体实现：
在 Timer 定时器中添加代码：
```c++
    //1. 普通绘图方式：
    CDC *pDC = GetDC();
    pDC->FillSolidRect(100, 50, 200, 100, RGB(100, 100, 100));

    pDC->SetTextColor(RGB(0, 0, 0));
    pDC->SetBkMode(TRANSPARENT);
    pDC->TextOut(110, 60, _T("ABCDEFGHIJKLMN"));
    pDC->TextOut(110, 80, _T("ABCDEFGHIJKLMN"));
    pDC->TextOut(110, 100, _T("ABCDEFGHIJKLMN"));
    pDC->TextOut(110, 120, _T("ABCDEFGHIJKLMN"));

    ReleaseDC(pDC);

    //2. 双缓冲绘图方式：
    CDC *pDC = GetDC();
    CDC memDC;
    memDC.CreateCompatibleDC(pDC);

    CBitmap bmp;
    bmp.CreateCompatibleBitmap(pDC, 200, 100);
    memDC.SelectObject(bmp);

    memDC.FillSolidRect(0, 0, 200, 100, RGB(100, 100, 100));
    memDC.SetTextColor(RGB(0, 0, 0));
    memDC.SetBkMode(TRANSPARENT);
    memDC.TextOut(10, 10, _T("ABCDEFGHIJKLMN"));
    memDC.TextOut(10, 30, _T("ABCDEFGHIJKLMN"));
    memDC.TextOut(10, 50, _T("ABCDEFGHIJKLMN"));
    memDC.TextOut(10, 70, _T("ABCDEFGHIJKLMN"));

    pDC->BitBlt(100, 50, 200, 100, &memDC, 0, 0, SRCCOPY);

    bmp.DeleteObject();
    memDC.DeleteDC();
    ReleaseDC(pDC);
```
 
3. 对话框贴图 -> 背景贴图：
* 将对话框的 Title Bar 属性置成 False；
* 将对话框的 Border 属性置成 Thin
* 插入背景图片资源，ID为：IDB_BK_IMG
     响应 WM_ERASEBKGND 消息进行图片的加载及背景的绘制：
```c++
BOOL CDrawTestDlg::OnEraseBkgnd(CDC* pDC)
{
    CDC memDC;
    memDC.CreateCompatibleDC(pDC);

    BITMAP bmp;
    CBitmap bkImg;
    bkImg.LoadBitmap(IDB_BK_IMG);
    bkImg.GetBitmap(&bmp);
    memDC.SelectObject(&bkImg);

    //SetWindowPos(NULL, 0, 0, bmp.bmWidth, bmp.bmHeight, SWP_NOMOVE|SWP_NOZORDER);
    CRect rect;
    GetClientRect(&rect);
    pDC->StretchBlt(0, 0, rect.Width(), rect.Height(), &memDC, 0, 0, bmp.bmWidth, bmp.bmHeight, SRCCOPY);

    memDC.DeleteDC();

    SetWindowText(_T("金山毒霸专杀工具"));
    return TRUE;

    return CDialog::OnEraseBkgnd(pDC);
}
```

4. 关闭按钮位图的设置：使用 CBitmapButton 类
* 前提：将按钮的 Owner Draw 属性置成 True；
* 绑定 CBitmapButton 类型的控件类型变量；
* 插入图片资源以及初始化代码：
```c++
    m_closeBtn.LoadBitmaps(IDB_CLOSE_NORMAL, IDB_CLOSE_DOWN);
    m_homeBtn.LoadBitmaps(IDB_HOME_NORMAL, IDB_HOME_DOWN);
    m_browseBtn.LoadBitmaps(IDB_BROWSE_NORMAL, IDB_BROWSE_DOWN);
    m_startBtn.LoadBitmaps(IDB_START_NORMAL, IDB_START_DOWN);

    响应函数的添加：
    ShellExecute(this->m_hWnd, _T("open"), _T("http://www.cctry.com"), _T(""), _T(""), SW_SHOW);

    //Start按钮
    void CDrawTestDlg::OnBnClickedStartBtn()
    {
        static BOOL m_bStart = TRUE;
        if(m_bStart)
        {
            m_startBtn.LoadBitmaps(IDB_OFF_NORMAL, IDB_OFF_DOWN);
            m_bStart = FALSE;
            m_startBtn.RedrawWindow();
            } else {
            m_startBtn.LoadBitmaps(IDB_START_NORMAL, IDB_START_DOWN);
            m_bStart = TRUE;
            m_startBtn.RedrawWindow();
        }
    }
```

6. 其他控件的添加：
```c++
    CListCtrl *pListCtrl = (CListCtrl *)GetDlgItem(IDC_LIST1);
    //DWORD dwStyle = pListCtrl->GetExtendedStyle();
    //dwStyle |= LVS_EX_FLATSB;
    //pListCtrl->SetExtendedStyle(dwStyle);

    pListCtrl->InsertColumn(0, _T("文件路径"), LVCFMT_LEFT, 380);
    pListCtrl->InsertColumn(1, _T("扫描结果"), LVCFMT_LEFT, 100);
    pListCtrl->InsertColumn(2, _T("状态"), LVCFMT_LEFT, 80);
```

7. 响应 WM_CTLCOLOR 消息，使 Static 静态文本框控件的背景置成透明：
```c++
HBRUSH CDrawTestDlg::OnCtlColor(CDC* pDC, CWnd* pWnd, UINT nCtlColor)
{
    HBRUSH hbr = CDialog::OnCtlColor(pDC, pWnd, nCtlColor);
    // TODO:  Change any attributes of the DC here
    if ( nCtlColor == CTLCOLOR_STATIC ) {
        pDC->SetBkMode(TRANSPARENT);
        return   (HBRUSH)::GetStockObject(NULL_BRUSH);
    }
    // TODO:  Return a different brush if the default is not desired
    return hbr;
}
```

8. 响应 WM_NCHITTEST 消息，模拟窗口拖动：
```c++
LRESULT CDrawTestDlg::OnNcHitTest(CPoint point)
{
// TODO: Add your message handler code here and/or call default
    CRect rect;
    GetClientRect(&rect);
    ClientToScreen(&rect);
    return rect.PtInRect(point) ? HTCAPTION : CDialog::OnNcHitTest(point);

    return CDialog::OnNcHitTest(point);
}
```
