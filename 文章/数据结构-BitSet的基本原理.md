###### BitSet的基本原理  

最后再了解一下BitSet的基本原理，BitSet是位操作的对象，值只有0或1，内部实现是一个long数组，初始只有一个long数组，所以BitSet最小的size是64，当存储的数据增加，初始化的Long数组已经无法满足时，BitSet内部会动态扩充，最终内部是由N个long来存储，BitSet的内部扩充和List，Set，Map等得实现差不多，而且都是对于用户透明的。  

1G的空间，有 8*1024*1024*1024=8589934592bit，也就是可以表示85亿个不同的数。  

BitSet用1位来表示一个数据是否出现过，0为没有出现过，1表示出现过。在long型数组中的一个元素可以存放64个数组，因为Java的long占8个byte=64bit，具体的实现，看看源码：  

###### 首先看看set方法的实现：  

```java
public void set(int bitIndex) {
   if (bitIndex < 0)   //set的数不能小于0
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

   int wordIndex = wordIndex(bitIndex);//将bitIndex右移6位，这样可以保证每64个数字在long型数组中可以占一个坑。
   expandTo(wordIndex);

   words[wordIndex] |= (1L << bitIndex); // Restores invariants
   checkInvariants();
}
```

###### get命令实现：  

```java
public boolean get(int bitIndex) {
   if (bitIndex < 0)
       throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

   checkInvariants();

   int wordIndex = wordIndex(bitIndex);//和get一样获取数字在long型数组的那个位置。
   return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);//在指定long型数组元素中获取值。
}
```

###### BitSet容量动态扩展：  

```java
private void ensureCapacity(int wordsRequired) {
   if (words.length < wordsRequired) {
        // Allocate larger of doubled size or required size
        int request = Math.max(2 * words.length, wordsRequired);//默认是扩大一杯的容量，如果传入的数字大于两倍的，则以传入的为准。
        // wordsRequired = 传入的数值右移6位 + 1
        words = Arrays.copyOf(words, request);
        sizeIsSticky = false;
   }
}
```