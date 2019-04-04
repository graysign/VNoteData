```c++
BOOL SaveBitmapToFile(HBITMAP   hBitmap, CString szfilename)
{
    HDC hDC; //   设备描述表    
    int iBits; //   当前显示分辨率下每个像素所占字节数 
    WORD wBitCount; //   位图中每个像素所占字节数   
    DWORD dwPaletteSize = 0   ; //   定义调色板大小，  位图中像素字节大小  ， 
    //   位图文件大小  ，  写入文件字节数
    DWORD dwBmBitsSize   ;  
    DWORD dwDIBSize, dwWritten   ;  
    BITMAP Bitmap;      
    BITMAPFILEHEADER bmfHdr; //位图属性结构 
    BITMAPINFOHEADER bi; //位图文件头结构  
    LPBITMAPINFOHEADER lpbi;   //位图信息头结构   
    HANDLE fh,   hDib,   hPal,hOldPal = NULL   ; //指向位图信息头结构,定义文件，分配内存句柄，调色板句柄

    //计算位图文件每个像素所占字节数 
    hDC         =     CreateDC(_T("DISPLAY"), NULL, NULL, NULL); 
    iBits       =     GetDeviceCaps(hDC, BITSPIXEL) * GetDeviceCaps(hDC, PLANES)  ;  
    DeleteDC(hDC)   ;  
    if (iBits <= 1)
    {
        wBitCount = 1;
    }
    else if (iBits <= 4)
    {
        wBitCount = 4;
    }
    else if (iBits <= 8)
    {
        wBitCount = 8;
    }
    else if (iBits <= 24)
    {
        wBitCount = 24;
    }
    else if (iBits <= 32)
    {
        wBitCount = 32;
    }  
    //计算调色板大小 
    if (wBitCount <= 8)
    {
        dwPaletteSize = (1 << wBitCount) * sizeof(RGBQUAD)   ;
    } 

    //设置位图信息头结构 
    GetObject(hBitmap, sizeof(BITMAP), (LPSTR)&Bitmap)   ;  
    bi.biSize          =     sizeof(BITMAPINFOHEADER);  
    bi.biWidth         =     Bitmap.bmWidth;  
    bi.biHeight        =     Bitmap.bmHeight;   
    bi.biPlanes        =     1;  
    bi.biBitCount      =     wBitCount;     
    bi.biCompression   =     BI_RGB;  
    bi.biSizeImage     =     0;  
    bi.biXPelsPerMeter =     0;  
    bi.biYPelsPerMeter =     0;  
    bi.biClrUsed       =     0;  
    bi.biClrImportant  =     0;   
    dwBmBitsSize       =     ((Bitmap.bmWidth * wBitCount + 31) / 32) * 4 * Bitmap.bmHeight   ;
    //为位图内容分配内存 
    hDib               =     GlobalAlloc(GHND, dwBmBitsSize + dwPaletteSize + sizeof(BITMAPINFOHEADER))   ;   
    lpbi               =     (LPBITMAPINFOHEADER)GlobalLock(hDib)   ;
    *lpbi              =     bi;  
    //   处理调色板  
    hPal               =     GetStockObject(DEFAULT_PALETTE);  
    if (hPal)
    {
        hDC = ::GetDC(NULL);  
        hOldPal = SelectPalette(hDC, (HPALETTE)hPal, FALSE)   ;  
        RealizePalette(hDC)   ;
    }  
    //   获取该调色板下新的像素值 
    GetDIBits(hDC, hBitmap, 0, (UINT)Bitmap.bmHeight, (LPSTR)lpbi + sizeof(BITMAPINFOHEADER) + dwPaletteSize, (LPBITMAPINFO)lpbi, DIB_RGB_COLORS);   
    //恢复调色板   
    if (hOldPal)
    {
        SelectPalette(hDC, (HPALETTE)hOldPal, TRUE); 
        RealizePalette(hDC)   ;   
        ::ReleaseDC(NULL, hDC)   ;
    } 
    //创建位图文件
    fh                 =     CreateFile(szfilename, GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_SEQUENTIAL_SCAN, NULL)   ; 
    if (fh == INVALID_HANDLE_VALUE)
    {
        return   false;
    }     
    //   设置位图文件头 
    bmfHdr.bfType         =      0x4D42;  //   "BM" 
    dwDIBSize             =      sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + dwPaletteSize + dwBmBitsSize;    
    bmfHdr.bfSize         =      dwDIBSize;  
    bmfHdr.bfReserved1    =      0;  
    bmfHdr.bfReserved2    =      0;  
    bmfHdr.bfOffBits      =      (DWORD)sizeof(BITMAPFILEHEADER) + (DWORD)sizeof(BITMAPINFOHEADER) + dwPaletteSize;  
    //   写入位图文件头 
    WriteFile(fh, (LPSTR)&bmfHdr, sizeof(BITMAPFILEHEADER), &dwWritten, NULL);  
    //   写入位图文件其余内容 
    WriteFile(fh, (LPSTR)lpbi, dwDIBSize, &dwWritten, NULL);  
    //消除内存分配     
    GlobalUnlock(hDib);   
    GlobalFree(hDib);  
    CloseHandle(fh);  
    return   true;
}
```