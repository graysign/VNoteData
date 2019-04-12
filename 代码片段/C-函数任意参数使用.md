例：求任意个自然数的平方和：  

```c
int SqSum(int n1, ...)
{
 va_list arg_ptr;
 int nSqSum = 0, n = n1;
 va_start(arg_ptr, n1); 
 
 while (n > 0)
 {
      nSqSum += (n * n);
      n = va_arg(arg_ptr, int);
 }
 va_end(arg_ptr);
 
 return nSqSum;
}
```

调用时  
```c
int nSqSum = SqSum(7, 2, 7, 11, -1);
```