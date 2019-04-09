# GDI+ 函数使用例子
一、通过Gdi+加载和显示PNG，JPG等格式的图片
```c++

//直接加载外部的图像

Image* image = new Image(L"test.png");  
//如果需要通过ID 来加载的话

BOOL CSmalltmpdemoDlg::ImageFromIDResource(UINT nID, LPCTSTR sTR, Image * & pImg)  
{  
    HINSTANCE hInst = AfxGetResourceHandle();  
    HRSRC hRsrc = ::FindResource (hInst,MAKEINTRESOURCE(nID),sTR); // type
    if (!hRsrc)  
        return FALSE;  
    // load resource into memory
    DWORD len = SizeofResource(hInst, hRsrc);  
    BYTE* lpRsrc = (BYTE*)LoadResource(hInst, hRsrc);  
    if (!lpRsrc)  
        return FALSE;  
    // Allocate global memory on which to create stream
    HGLOBAL m_hMem = GlobalAlloc(GMEM_FIXED, len);  
    BYTE* pmem = (BYTE*)GlobalLock(m_hMem);
    memcpy(pmem,lpRsrc,len);
    IStream* pstm;
    CreateStreamOnHGlobal(m_hMem,FALSE,&pstm);
    // load from stream
    pImg=Gdiplus::Image::FromStream(pstm);
    // free/release stuff
    GlobalUnlock(m_hMem);
    pstm-<Release();
    FreeResource(lpRsrc);
    return TRUE;
}

//调用方式

Image * pImage = NULL;
ImageFromIDResource(IDR_PNG_NO_PIC, L"png", pImage);
delete pImage;
/////////////////////////////////////////////////////////////////////////
Image * pImage = NULL;
ImageFromIDResource(IDR_PNG_NO_PIC, L"jpg", pImage);
delete pImage;
//////////////////////////////////////////////////////////////////////////
Image * pImage = NULL; 
ImageFromIDResource(IDR_PNG_NO_PIC, L"bitmap", pImage);
delete pImage;
```

二、实现一个渐变的画刷 
```c++

CClientDC dc(this);
CRect rect;
//获得当前客户区的大小
GetClientRect(&rect);
//创建Graphics对象
Graphics graphics(dc);
//创建渐变画刷
LinearGradientBrush lgb(Point(0, 0), Point(rect.right, rect.bottom), Color::Blue, Color::Green);  
//填充
graphics.FillRectangle(&lgb, 0, 0, rect.right, rect.bottom);  
```