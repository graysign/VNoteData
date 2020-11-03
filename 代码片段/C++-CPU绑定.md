# Linux  
cpu亲和性（affinity）用于绑定到固定的CPU上。主要有以下函数。  

```c++
int sched_setaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);//进程绑定函数
int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize, const cpu_set_t *cpuset);//线程绑定函数
void CPU_ZERO (cpu_set_t *set)  /*这个宏对 CPU 集 set 进行初始化，将其设置为空集。*/
void CPU_SET (int cpu, cpu_set_t *set) /*这个宏将 指定的 cpu 加入 CPU 集 set 中*/
void CPU_CLR (int cpu, cpu_set_t *set) /*这个宏将 指定的 cpu 从 CPU 集 set 中删除。*/
int CPU_ISSET (int cpu, const cpu_set_t *set)
/*如果 cpu 是 CPU 集 set 的一员，这个宏就返回一个非零值（true），否则就返回零（false）。*/
int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize, const cpu_set_t *cpuset);
```
## 1.获取核心数量  

```c++
#include <unistd.h>

int sysconf(_SC_NPROCESSORS_CONF);/* 返回系统可以使用的核数，但是其值会包括系统中禁用的核的数目，因 此该值并不代表当前系统中可用的核数 */
int sysconf(_SC_NPROCESSORS_ONLN);/* 返回值真正的代表了系统当前可用的核数 */

/* 以下两个函数与上述类似 */
#include <sys/sysinfo.h>

int get_nprocs_conf (void);/* 可用核数 */
int get_nprocs (void);/* 真正的反映了当前可用核数 */
```

## 2.进程绑定到一个特定的CPU  

```c++
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sched.h>
/* 设置进程号为pid的进程运行在mask所设定的CPU上
 * 第二个参数cpusetsize是mask所指定的数的长度
 * 通常设定为sizeof(cpu_set_t)
 * 如果pid的值为0,则表示指定的是当前进程 
 */
int sched_setaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);
int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);/* 获得pid所指示的进程的CPU位掩码,并将该掩码返回到mask所指向的结构中 */
```
  
```c++
#include<stdlib.h>
#include<stdio.h>
#include<sys/types.h>
#include<sys/sysinfo.h>
#include<unistd.h>

#define __USE_GNU
#include<sched.h>
#include<ctype.h>
#include<string.h>
#include<pthread.h>
#define THREAD_MAX_NUM 200  //1个CPU内的最多进程数

int num=0;  //cpu中核数
void* threadFun(void* arg)  //arg  传递线程标号（自己定义）
{
         cpu_set_t mask;  //CPU核的集合
         cpu_set_t get;   //获取在集合中的CPU
         int *a = (int *)arg; 
         int i;

         printf("the thread is:%d\n",*a);  //显示是第几个线程
         CPU_ZERO(&mask);    //置空
         CPU_SET(*a,&mask);   //设置亲和力值
         if (sched_setaffinity(0, sizeof(mask), &mask) == -1)//设置线程CPU亲和力
         {
                   printf("warning: could not set CPU affinity, continuing...\n");
         }

           CPU_ZERO(&get);
           if (sched_getaffinity(0, sizeof(get), &get) == -1)//获取线程CPU亲和力
           {
                    printf("warning: cound not get thread affinity, continuing...\n");
           }
           for (i = 0; i < num; i++)
           {
                    if (CPU_ISSET(i, &get))//判断线程与哪个CPU有亲和力
                    {
                             printf("this thread %d is running processor : %d\n", i,i);
                    }
           }

         return NULL;
}

int main(int argc, char* argv[])
{
         int tid[THREAD_MAX_NUM];
         int i;
         pthread_t thread[THREAD_MAX_NUM];

         num = sysconf(_SC_NPROCESSORS_CONF);  //获取核数
         if (num > THREAD_MAX_NUM) {
            printf("num of cores[%d] is bigger than THREAD_MAX_NUM[%d]!\n", num, THREAD_MAX_NUM);
            return -1;
         }
         printf("system has %i processor(s). \n", num);
         for(i=0;i<num;i++)
         {
                   tid[i] = i;  //每个线程必须有个tid[i]
                   pthread_create(&thread[i],NULL,threadFun,(void*)&tid[i]);
         }
         for(i=0; i< num; i++)
         {
                   pthread_join(thread[i],NULL);//等待所有的线程结束，线程为死循环所以CTRL+C结束
         }
         return 0;
}
```

## 3.线程绑定到一个特定的CPU  

```c++
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <pthread.h>

int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize, const cpu_set_t *cpuset);
int pthread_getaffinity_np(pthread_t thread, size_t cpusetsize, cpu_set_t *cpuset);
Compile and link with -pthread.
```

```c++
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

#define handle_error_en(en, msg) \
        do { errno = en; perror(msg); exit(EXIT_FAILURE); } while (0)

int
main(int argc, char *argv[])
{
    int s, j;
    cpu_set_t cpuset;
    pthread_t thread;

    thread = pthread_self();

    /* Set affinity mask to include CPUs 0 to 7 */

    CPU_ZERO(&cpuset);
    for (j = 0; j < 8; j++)
        CPU_SET(j, &cpuset);

    s = pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
    if (s != 0)
        handle_error_en(s, "pthread_setaffinity_np");

    /* Check the actual affinity mask assigned to the thread */
    s = pthread_getaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
    if (s != 0)
        handle_error_en(s, "pthread_getaffinity_np");

    printf("Set returned by pthread_getaffinity_np() contained:\n");
    for (j = 0; j < CPU_SETSIZE; j++)
        if (CPU_ISSET(j, &cpuset))
            printf("    CPU %d\n", j);

    exit(EXIT_SUCCESS);
}
```





# Windows  

## 1.获取核心数量  

```c++
SYSTEM_INFO info;
GetSystemInfo(&info);
printf("Number of processors: %d.\n", info.dwNumberOfProcessors);
```

## 2.进程绑定到一个特定的CPU  

```c++
BOOL SetProcessAffinityMask(
  HANDLE hProcess,
  DWORD_PTR dwProcessAffinityMask
);

```

```c++
// get system info
    SYSTEM_INFO SystemInfo;
    GetSystemInfo( & SystemInfo);
    printf( " "
         " dwNumberOfProcessors=%u, dwActiveProcessorMask=%u, wProcessorLevel=%u, "
         " wProcessorArchitecture=%u, dwPageSize=%u " ,
        SystemInfo.dwNumberOfProcessors, SystemInfo.dwActiveProcessorMask, SystemInfo.wProcessorLevel, 
        SystemInfo.wProcessorArchitecture,SystemInfo.dwPageSize
        );
    if (SystemInfo.dwNumberOfProcessors  <=   1 )  return ;

    DWORD dwMask  =   0x0000 ;
    DWORD dwtmp  =   0x0001 ;
    int  nProcessorNum  =   0 ;
    for ( int  i  =   0 ; i  <   32 ; i ++ ) {
        if(SystemInfo.dwActiveProcessorMask & dwtmp){
            nProcessorNum++;
            if(nProcessorNum <= 2){
                //如果系统中有多个处理器，则选择第二个处理器
                dwMask = dwtmp;
            }else{
                break;
            }
        }
        dwtmp *= 2;
    } // end of for
     // 进程与指定cpu绑定
     SetProcessAffinityMask(GetCurrentProcess(), dwMask);
     // 线程与指定cpu绑定
     // SetThreadAffinityMask(GetCurrentThread(),dwMask);
     return  ;
```
## 3.线程绑定到一个特定的CPU  

```c++
DWORD_PTR SetThreadAffinityMask(HANDLE hThread, DWORD_PTR dwThreadAffinityMask);
第一个参数为线程句柄:GetCurrentThread()
第二个参数为一个mask:可取值为0~2^31(32位）和0~2^63（64位），每一位代表每一个CPU是否使用
你要指定进程到第0个CPU上，则mask=0×01
第1个CPU：mask=0×02
第2个CPU：mask=0×04 （注意不是0×03）
第3个CPU：mask=0×08

```