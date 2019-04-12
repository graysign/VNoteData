如果有一个庞大的字符串数组，然后给你一个单独的字符串，让你从这个数组中查找是否有这个字符串并找到它，你会怎么做？  

* 老老实实从头查到尾，一个一个比较，直到找到为止
* 最合适的算法自然是使用HashTable（哈希表）  

###### 定义
一般是一个整数，通过某种算法，可以把一个字符串"压缩" 成一个整数，这个数称为Hash，当然，无论如何，一个32位整数是无法对应回一个字符串的，但在程序中，两个字符串计算出的Hash值相等的可能非常小.

###### MPQ中的Hash算法    

```c
unsigned long HashString(char *lpszFileName, unsigned long dwHashType)
{ 
     unsigned char *key = (unsigned char *)lpszFileName;
    unsigned long seed1 = 0x7FED7FED, seed2 = 0xEEEEEEEE;
    int ch;
    while(*key != 0)
    { 
          ch = toupper(*key++);
        seed1 = cryptTable[(dwHashType << 8) + ch] ^ (seed1 + seed2);
        seed2 = ch + seed1 + seed2 + (seed2 << 5) + 3; 
    }
    return seed1; 
} 
```

Blizzard的这个算法是非常高效的，被称为"One-Way Hash"（所谓One-Way Hash,就是无法从求得的hash值通过简单的逆运算就得到原来的字符串。类似与加密算法中的message digest）。
  
是不是把第一个算法改进一下，改成逐个比较字符串的Hash值就可以了呢，答案是，远远不够，要想得到最快的算法，就不能进行逐个的比较，通常是构造一个哈希表(Hash Table)来解决问题，哈希表是一个大数组，这个数组的容量根据程序的要求来定义，例如1024，每一个Hash值通过取模运算 (mod)对应到数组中的一个位置，这样，只要比较这个字符串的哈希值对应的位置又没有被占用，就可以得到最后的结果了，想想这是什么速度？是的，是最快的O(1)，现在仔细看看这个算法吧  

```c
int GetHashTablePos(char *lpszString, SOMESTRUCTURE *lpTable, int nTableSize)
{ 
     int nHash = HashString(lpszString), nHashPos = nHash % nTableSize;
    if (lpTable[nHashPos].bExists && !strcmp(lpTable[nHashPos].pString, lpszString)) 
      return nHashPos; 
    else 
      return -1; //Error value 
} 
```

看到此，我想大家都在想一个很严重的问题："如果两个字符串在哈希表中对应的位置相同怎么办？",毕竟一个数组容量是有限的，这种可能性很大。解决该问题的方法很多，我首先想到的就是用"链表",感谢大学里学的数据结构教会了这个百试百灵的法宝，我遇到的很多算法都可以转化成链表来解决，只要在哈希表的每个入口挂一个链表，保存所有对应的字符串就OK了。  


事情到此似乎有了完美的结局，如果是把问题独自交给我解决，此时我可能就要开始定义数据结构然后写代码了。然而Blizzard的程序员使用的方法则是更精妙的方法。基本原理就是：他们在哈希表中不是用一个哈希值而是用三个哈希值来校验字符串。  

中国有句古话"再一再二不能再三再四"，看来Blizzard也深得此话的精髓，如果说两个不同的字符串经过一个哈希算法得到的入口点一致有可能，但用三个不同的哈希算法算出的入口点都一致，那几乎可以肯定是不可能的事了，这个几率是1:18889465931478580854784，大概是10的 22.3次方分之一，对一个游戏程序来说足够安全了。
现在再回到数据结构上，Blizzard使用的哈希表没有使用链表，而采用"顺延"的方式来解决问题，看看这个算法：  
  
```c
int GetHashTablePos(char *lpszString, MPQHASHTABLE *lpTable, int nTableSize)
{ 
 const int HASH_OFFSET = 0, HASH_A = 1, HASH_B = 2;
int nHash = HashString(lpszString, HASH_OFFSET);
int nHashA = HashString(lpszString, HASH_A);
int nHashB = HashString(lpszString, HASH_B);
int nHashStart = nHash % nTableSize, nHashPos = nHashStart;
while (lpTable[nHashPos].bExists)
{ //比较的是Table中存储的另外两个Hash函数的值，Table中不存储字符串
  if (lpTable[nHashPos].nHashA == nHashA && lpTable[nHashPos].nHashB == nHashB) 
   return nHashPos; 
  else  //冲突处理
   nHashPos = (nHashPos + 1) % nTableSize;
  
  if (nHashPos == nHashStart) 
   break; 
 }
return -1; //Error value 
} 
```

1. 计算出字符串的三个哈希值（一个用来确定位置，另外两个用来校验)
2. 察看哈希表中的这个位置
3. 哈希表中这个位置为空吗？如果为空，则肯定该字符串不存在，返回
4. 如果存在，则检查其他两个哈希值是否也匹配，如果匹配，则表示找到了该字符串，返回
5. 移到下一个位置，如果已经越界，则表示没有找到，返回
6. 看看是不是又回到了原来的位置，如果是，则返回没找到
7. 回到3

怎么样，很简单的算法吧，但确实是天才的idea, 其实最优秀的算法往往是简单有效的算法，Blizzard被称为最卓越的游戏制作公司，不愧于此。(转载注：这种解决hash collision的方法相对于用linked list方法的缺点在于，hash表的entry只能代表一个字符串，如果hash表满了则无法在向hash表中加入新的entry)