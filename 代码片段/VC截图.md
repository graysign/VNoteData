
```c++
HBITMAP CopyScreenToBitmap(int x1, int x2, int y1, int y2)
{ 
    HDC hSrcDC, hMemDC;//屏幕和内存设备描述表
    HBITMAP hBitmap, hOldBitmap;//位图句柄
    int nX, nY, nX2, nY2;//选定区域坐标
    int nWidth, nHeight;//位图宽度和高度
    int xScrn, yScrn;//屏幕分辨率
    /*
    //确保选定区域不为空矩形
    if(IsRectEmpty(lpRect))
    return NULL;*/
    //为屏幕创建设备描述表
    hSrcDC     =     CreateDC("DISPLAY", NULL, NULL, NULL);
    //为屏幕设备描述表创建兼容的内存设备描述表
    hMemDC     =     CreateCompatibleDC(hSrcDC);
    //获得选定区域坐标
    nX         =     x1;
    nY         =     y1;
    nX2        =     x2;
    nY2        =     y2;
    //获得屏幕分辨率
    xScrn      =     GetDeviceCaps(hSrcDC, HORZRES);
    yScrn      =     GetDeviceCaps(hSrcDC, VERTRES);
    //确保选定区域是可见的
    if(nX<0)
            nX = 0;
    if(nY<0)
            nY = 0;
    if(nX2>xScrn)
            nX2 = xScrn;
    if(nY2>yScrn)
            nY2 = yScrn;
    nWidth      = nX2 - nX;
    nHeight     = nY2 - nY;
    //创建一个与屏幕设备描述表兼容的位图
    hBitmap     =     CreateCompatibleBitmap(hSrcDC, nWidth, nHeight);
    //把新位图选到内存设备描述表中
    hOldBitmap  =     (HBITMAP)SelectObject(hMemDC, hBitmap);
    //把屏幕设备描述表拷贝到内存设备描述表中
    BitBlt(hMemDC, 0, 0, nWidth, nHeight, hSrcDC, nX, nY, SRCCOPY);
    //得到屏幕位图的句柄
    hBitmap     =     (HBITMAP)SelectObject(hMemDC, hOldBitmap);
    //清除
    DeleteDC(hSrcDC);
    DeleteDC(hMemDC);
    //返回位置句柄
    return hBitmap;
}
```

