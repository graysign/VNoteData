如何从一个字体文件中读取出需要的信息呢？比如字体名称（非文件名）、字体版权方等等，刚好最学习了浏览器里面的二进制，就尝试了用浏览器来解析字体文件信息。  

首先介绍两种常见的字体文件，ttf(TrueType Font)与ttc(TrueType Collection)  

说白了，ttf表示单一字体，而ttc是多个ttf的合集，我们弄懂了ttf，那么ttc就非常简单了。  

# ttf  

ttf文件主要由三大部分组成：  

`Header + N个Table Directory + N个Table Header + M个Table Record`

我们以C类型的结构体形式表示各结构：  


## Header  

```c
typedef struct _tagTT_OFFSET_TABLE{
  uint16 uMajorVersion,
  uint16 uMinorVersion
  uint16 uNumOfTables, // 关注它--> Table的个数
  uint16 uSearchRange,
  uint16 uEntrySelector,
  uint16 uRangeShift
}TTF_HEADER_TABLE;
```

Header总共占据了2*6 = 12，我们从文件最开始读取12个字节即可得到如上信息，其中若uMajorVersion不为1且uMinorVersion不为0，我们可以认为该文件不为合法的ttf文件。  

紧接着，我们关注uNumOfTables的值，其决定了接下来会有多少个连续的Table Directory。  

我们来看看Table Directory的结构：  

## Table Directory  

```c
typedef struct _tagTT_TABLE_DIRECTORY{
  char szTag[4]; // 关注它 --> Table Name
  ULONG uCheckSum; // 校验和
  ULONG uOffset; // 关注它--> 对应的Table的绝对位置
  ULONG uLength; // 对应的Table长度
}TT_TABLE_DIRECTORY;
```

单个Table Directory的长度为4*3=12字节，从Header的末尾开始，按长度12为单位，循环读取至多uNumOfTables次，我们就可以读取到所有的Table Directory了。当然，本文只打算读取字体的基本信息，所以，我们只关注szTag为name的Table Directory。  

一旦找到需要的Table Directory,那么就可以根据其uOffset的值找到Name Table Header了，先看看Table Header的结构：  


## Table Header  

```c
typedef struct _tagTT_NAME_TABLE_HEADER{
  uint16 uFSelector; // 始终为0
  uint16 uNRCount; // 关注它--> Name Record 的数量
  uint16 uStorageOffset; // Name Record相对于该结构起始位置的偏移值
}TT_NAME_TABLE_HEADER;
```

通过uStorageOffset，我们即可从uOffset+uStorageOffset开始读取uNRCount个连续的Name Record，其结构为：

```c
typedef struct _tagTT_NAME_RECORD{
  uint16 uPlatformID;
  uint16 uEncodingID;
  uint16 uLanguageID;
  uint16 uNameID; // name id，我们暂时只取0-7的
  uint16 uStringLength;
  uint16 uStringOffset; //from start of storage area
}TT_NAME_RECORD;
```

每个Name Record的长度为2*6=12，我们要读取的信息为uNameID范围0-7(更多)，分别对应如下：  

```c
{
    0: 'copyright',
    1: 'fontFamily',
    2: 'fontSubFamily',
    3: 'fontIdentifier',
    4: 'fontName',
    5: 'fontVersion',
    6: 'postscriptName',
    7: 'trademark',
}
```

另外考虑读取的编码为Unicode情况，要求：`uPlatformID === 0 && uEncodingID === (0,1,3)` 或 `uPlatformID === 3 && uEncodingID === 1`  

满足如上条件后，我们通过uOffset+uStorageOffset+uStringOffset分别获取每条Name Record中的字符串值，即可对应到上面列出的八个属性的值。  

到这里，我们就得到了单个ttf文件中需要的信息了。  


# ttc是由多个ttf文件构成  

## header结构为：  

```c
typedef struct _tagTTC_HEADER_LE {
   uint32 tag;
   uint16 uMajorVersion;
   uint16 uMinorVersion;
   uint32 uNumFonts;
} TTC_HEADER_LE;
```

其中tag固定为ttcf,uNumFonts表明该文件包含了多少个ttf字体，同时，在紧挨着该结构后面是uNumFonts个4字节的offset，用于表示每个ttf相对于文件起始位置的便宜。一旦找到这个偏移，读取每个ttf的方式就和前面一样了。  

完整实现代码见[github](https://github.com/Elity/FontReader)