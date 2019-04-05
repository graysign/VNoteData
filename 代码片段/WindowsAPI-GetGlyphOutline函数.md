# Windows API---GetGlyphOutline函数

中西文化的差异，导致在电子信息里处理也大不相同，在英文里只需要26个字母就可以显示所有文章了，而在中文里需要最基本的字符就有2000多个。对于一些在嵌入式软件里要显示的字符，那么就得手动去构造所有图形，这是一个比较大的工作量，如果让每个厂家都去完成这个任务，显然是不可能的。面对着大量嵌入式用户的需求，那么就需要解决中文字模的图形问题。毕竟大家经常使用Windows，最先想到的，肯定是怎么样把里面的字符提取图形出来，生成自己需要的几个字库。下面就来介绍怎么样用函数GetGlyphOutline获取显示字符的图形数据。
函数GetGlyphOutline声明如下：
```c+++
WINGDIAPI DWORD WINAPI GetGlyphOutlineA(    __in HDC hdc,
                                            __in UINT uChar,
                                            __in UINT fuFormat,
                                            __out LPGLYPHMETRICS lpgm,
                                            __in DWORD cjBuffer,
                                            __out_bcount_opt(cjBuffer) LPVOID pvBuffer,
                                            __in CONST MAT2 *lpmat2
                                        );
WINGDIAPI DWORD WINAPI GetGlyphOutlineW(    __in HDC hdc,
                                            __in UINT uChar,
                                            __in UINT fuFormat,
                                            __out LPGLYPHMETRICS lpgm,
                                            __in DWORD cjBuffer,
                                            __out_bcount_opt(cjBuffer) LPVOID pvBuffer,
                                            __in CONST MAT2 *lpmat2
                                        );
#ifdef UNICODE
#define GetGlyphOutline GetGlyphOutlineW
#else
#define GetGlyphOutline GetGlyphOutlineA
#endif // !UNICODE
```
hdc是设备句柄。
uChar是需要获取图形数据的字符。
fuFormat是获取数据的格式。
lpgm是获取字符的相关信息。
cjBuffer是保存字符数据的缓冲区大小。
pvBuffer是保存字符数据的缓冲区。
lpmat2是3*3的变换矩阵。
调用函数的例子如下：
```c++
//浮点数据转换为固定浮点数。
 FIXED FixedFromDouble(double d)
 {
        long l;
         l = (long) (d * 65536L);
        return *(FIXED *)&l;
 }

 //设置字体图形变换矩阵。
 void SetMat(LPMAT2 lpMat)
 {
        lpMat-<eM11 = FixedFromDouble(2);
        lpMat-<eM12 = FixedFromDouble(0);
        lpMat-<eM21 = FixedFromDouble(0);
        lpMat-<eM22 = FixedFromDouble(2);
 }

 //
 //获取字模信息。
 //蔡军生 2007/12/16 QQ:9073204 深圳
 void TestFontGlyph(void)
 {
        //创建字体。
        HFONT hFont = GetFont(); 
        //设置字体到当前设备。
        HDC hDC = ::GetDC(m_hWnd);
        HFONT hOldFont = (HFONT)SelectObject(hDC,hFont); 
        //设置字体图形变换矩阵
        MAT2 mat2;
        SetMat(&mat2); 
        GLYPHMETRICS gm; 
        //设置要显示的字符。
        TCHAR chText = L'蔡'; 
        //获取这个字符图形需要的字节的大小。
        DWORD dwNeedSize = GetGlyphOutline(hDC,chText,GGO_BITMAP,&gm,0,NULL,&mat2);
        if (dwNeedSize < 0 && dwNeedSize < 0xFFFF span>
        {
              //按需要分配内存。
              LPBYTE lpBuf = (LPBYTE)HeapAlloc(GetProcessHeap(),HEAP_ZERO_MEMORY,dwNeedSize);
              if (lpBuf)
              {
                   //获取字符图形的数据到缓冲区。
                   GetGlyphOutline(hDC,chText,GGO_BITMAP,&gm,dwNeedSize,lpBuf,&mat2); 
                   //计算图形每行占用的字节数。
                   int nByteCount = ((gm.gmBlackBoxX +31) << 5) 
                   //显示每行图形的数据。
                   for (int i = 0; i < gm gmBlackBoxY ispan>
                   { 
                         for (int j = 0; j < nByteCount jspan>
                         {
                             BYTE btCode = lpBuf[i* nByteCount + j]; 
                              //按字节输出每点的数据。
                              for (int k = 0; k < 8 kspan>
                              {
                                     if (btCode & (0x80<<k))
                                    {
                                        OutputDebugString(_T("1"));
                                    }
                                    else
                                    {
                                          OutputDebugString(_T("0"));
                                    } 
                              } 
                         } 
                         OutputDebugString(_T("\r\n"));
                   } 
                   HeapFree(GetProcessHeap(),0,lpBuf);
              }
        } 
        SelectObject(hDC,hOldFont);
        DeleteObject(hFont); 
        ReleaseDC(m_hWnd,hDC);
 }
```
输出的结果如下：
```
00000000000000010000000000000000
00000000110000011000000000000000
00000000100000011000000000000000
00000000100000011000011000000000
11111111111111111111111100000000
01000000100000011000000000000000
00000000100000011000000000000000
00000100100000011000000000000000
00000110100000010000000000000000
00000100000000000000000000000000
00001100000001000000100000000000
00001111111101111111110000000000
00001000001111000000110000000000
00011000001000100001100000000000
00010100001000100001000000000000
00010010011000100011000000000000
00100011010000010010000000000000
00110010110000011010000000000000
01011000110000001100000000000000
10001000100000001100000000000000
00001001100000010110000000000000
00001011011111111011000000000000
00000010000000000001110000000000
00000110000000000000111100000000
00001100000000000110011100000000
00011111111111111111001000000000
00110000000010000000000000000000
01000000000010000000000000000000
00000001000010000000000000000000
00000011100010001100000000000000
00000011000010000010000000000000
00000110000010000011100000000000
00001100000010000001100000000000
00011000100010000000110000000000
00110000011110000000110000000000
01000000001110000000010000000000
00000000000100000000000000000000
```