# linux-c毫秒级sleep
```c
void Delay(int ms_count)
{
    struct timespec req;
    int i;

    req.tv_sec = 0;
    req.tv_nsec = 1000000; /* 1 ms */

    for (i = 0; i < ms_count; i++)
        nanosleep(&req, NULL);
}

```