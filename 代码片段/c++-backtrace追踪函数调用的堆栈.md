

一般察看函数运行时堆栈的方法是使用GDB之类的外部调试器,但是,有些时候为了分析程序的BUG,(主要针对长时间运行程序的分析),在程序出错时打印出函数的调用堆栈是非常有用的。  

在头文件"execinfo.h"中声明了三个函数用于获取当前线程的函数调用堆栈  

```c++
//该函数用与获取当前线程的调用堆栈,获取的信息将会被存放在buffer中,它是一个指针列表。
//参数 size 用来指定buffer中可以保存多少个void* 元素。
//函数返回值是实际获取的指针个数,最大不超过size大小，在buffer中的指针实际是从堆栈中获取的返回地址,每一个堆栈框架有一个返回地址
//注意某些编译器的优化选项对获取正确的调用堆栈有干扰,另外内联函数没有堆栈框架;删除框架指针也会使无法正确解析堆栈内容
int backtrace(void **buffer,int size)

//backtrace_symbols将从backtrace函数获取的信息转化为一个字符串数组
//参数buffer应该是从backtrace函数获取的数组指针,size是该数组中的元素个数(backtrace的返回值)
//函数返回值是一个指向字符串数组的指针,它的大小同buffer相同.每个字符串包含了一个相对于buffer中对应元素的可打印信息.它包括函数名，函数的偏移地址,和实际的返回地址。
//现在,只有使用ELF二进制格式的程序和苦衷才能获取函数名称和偏移地址.在其他系统,只有16进制的返回地址能被获取.另外,你可能需要传递相应的标志给链接器,以能支持函数名功能(比如,在使用GNU ld的系统中,你需要传递(-rdynamic))。
//该函数的返回值是通过malloc函数申请的空间,因此调用这必须使用free函数来释放指针.
//注意:如果不能为字符串获取足够的空间函数的返回值将会为NULL
char** backtrace_symbols (void *const *buffer, int size)


//backtrace_symbols_fd与backtrace_symbols 函数具有相同的功能,不同的是它不会给调用者返回字符串数组,而是将结果写入文件描述符为fd的文件中,每个函数对应一行.它不需要调用malloc函数,因此适用于有可能调用该函数会失败的情况。
void backtrace_symbols_fd (void *const *buffer, int size, int fd)
```
，
下面是一个使用backtrace捕获异常并打印函数调用堆栈的例子：

```c++
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <execinfo.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>
#define PRINT_DEBUG

#define MAX_STACK_LEVEL	64
static void LogBackTrace(int sig)
{
    void *array[MAX_STACK_LEVEL];
    size_t size = backtrace(array, MAX_STACK_LEVEL);
#ifdef PRINT_DEBUG
    char **strings;
    strings = backtrace_symbols(array, size);
    printf("Obtained %d stack frames.\n", size);
    for (int i = 0; i < size; i++)
        printf("%s\n", strings[i]);
    free(strings);
    /*char cmd[64] = "addr2line -C -f -e ";
    char *prog = cmd + strlen(cmd);
    readlink("/proc/self/exe", prog, sizeof(cmd) - strlen(cmd) - 1); // 获取进程的完整路径
    FILE *fp = popen(cmd, "w");
    if (fp != NULL)
    {
        for (i = 0; i < size; ++i)
        {
            fprintf(fp, "%p\n", array[i]);
        }
        pclose(fp);
    }*/
#else
    int fd = open("err.log", O_CREAT | O_WRONLY);
    backtrace_symbols_fd(array, size, fd);
    close(fd);
#endif
    exit(0);
}

void die()
{
    printf("die \n");
    char *test1;
    char *test2;
    char *test3;
    char *test4 = NULL;
    free((void*)1);
    strcpy(test4, "ab");
}

void test1()
{
    printf("test1 \n");
    die();
}

int main(int argc, char **argv)
{
    struct sigaction myAction;
    myAction.sa_handler = LogBackTrace;
    sigemptyset(&myAction.sa_mask);
    myAction.sa_flags = SA_RESTART | SA_SIGINFO;
    sigaction(SIGSEGV, &myAction, NULL); // 无效内存引用
    sigaction(SIGABRT, &myAction, NULL); // 异常终止
    test1();
    printf("main1 \n");
    printf("main2 \n");
    printf("main3 \n");
}

```


编译`g++ -g -rdynamic  backtrace.c    `

```console
如果不使用特殊的链接器选项，符号名称可能不可用。对于使用GNU链接器的系统，必须使用-rdynamic链接器选项。
注意“静态”函数的名称不公开，并且在回溯中也不可用。
```


```txt
BACKTRACE(3)               Linux Programmer’s Manual              BACKTRACE(3)

NAME
       backtrace, backtrace_symbols, backtrace_symbols_fd - support for application self-debugging

SYNOPSIS
       #include <execinfo.h>

       int backtrace(void **buffer, int size);

       char **backtrace_symbols(void *const *buffer, int size);

       void backtrace_symbols_fd(void *const *buffer, int size, int fd);

DESCRIPTION
       backtrace()  returns  a backtrace for the calling program, in the array pointed to by buffer.  A backtrace is the series of currently active function calls for the program.
       Each item in the array pointed to by buffer is of type void *, and is the return address from the corresponding stack frame.  The size argument specifies the maximum number
       of  addresses  that can be stored in buffer.  If the backtrace is larger than size, then the addresses corresponding to the size most recent function calls are returned; to
       obtain the complete backtrace, make sure that buffer and size are large enough.

       Given the set of addresses returned by backtrace() in buffer, backtrace_symbols() translates the addresses into an array of strings that  describe  the  addresses  symboli-
       cally.   The  size  argument  specifies the number of addresses in buffer.  The symbolic representation of each address consists of the function name (if this can be deter-
       mined), a hexadecimal offset into the function, and the actual return address (in hexadecimal).  The address of the array of string pointers is  returned  as  the  function
       result  of  backtrace_symbols().   This array is malloc(3)ed by backtrace_symbols(), and must be freed by the caller.  (The strings pointed to by the array of pointers need
       not and should not be freed.)

       backtrace_symbols_fd() takes the same buffer and size arguments as backtrace_symbols(), but instead of returning an array of strings to the caller, it writes  the  strings,
       one per line, to the file descriptor fd.  backtrace_symbols_fd() does not call malloc(3), and so can be employed in situations where the latter function might fail.

RETURN VALUE
       backtrace()  returns the number of addresses returned in buffer, which is not greater than size.  If the return value is less than size, then the full backtrace was stored;
       if it is equal to size, then it may have been truncated, in which case the addresses of the oldest stack frames are not returned.

       On success, backtrace_symbols() returns a pointer to the array malloc(3)ed by the call; on error, NULL is returned.

VERSIONS
       backtrace(), backtrace_symbols(), and backtrace_symbols_fd() are provided in glibc since version 2.1.

CONFORMING TO
       These functions are GNU extensions.

NOTES
       These functions make some assumptions about how a function’s return address is stored on the stack.  Note the following:

       *  Omission of the frame pointers (as implied by any of gcc(1)’s non-zero optimization levels) may cause these assumptions to be violated.

       *  Inlined functions do not have stack frames.

       *  Tail-call optimization causes one stack frame to replace another.

       The symbol names may be unavailable without the use of special linker options.  For systems using the GNU linker, it is necessary to use the -rdynamic linker option.   Note
       that names of "static" functions are not exposed, and won’t be available in the backtrace.

EXAMPLE
       The program below demonstrates the use of backtrace() and backtrace_symbols().  The following shell session shows what we might see when running the program:

           $ cc -rdynamic prog.c -o prog
           $ ./prog 3
           backtrace() returned 8 addresses
           ./prog(myfunc3+0x5c) [0x80487f0]
           ./prog [0x8048871]
           ./prog(myfunc+0x21) [0x8048894]
           ./prog(myfunc+0x1a) [0x804888d]
           ./prog(myfunc+0x1a) [0x804888d]
           ./prog(main+0x65) [0x80488fb]
           /lib/libc.so.6(__libc_start_main+0xdc) [0xb7e38f9c]
           ./prog [0x8048711]

   Program source
       #include <execinfo.h>
       #include <stdio.h>
       #include <stdlib.h>
       #include <unistd.h>

       void
       myfunc3(void)
       {
           int j, nptrs;
       #define SIZE 100
           void *buffer[100];
           char **strings;

           nptrs = backtrace(buffer, SIZE);
           printf("backtrace() returned %d addresses\n", nptrs);

           /* The call backtrace_symbols_fd(buffer, nptrs, STDOUT_FILENO)
              would produce similar output to the following: */

           strings = backtrace_symbols(buffer, nptrs);
           if (strings == NULL) {
               perror("backtrace_symbols");
               exit(EXIT_FAILURE);
           }

           for (j = 0; j < nptrs; j++)
               printf("%s\n", strings[j]);

           free(strings);
       }

       static void   /* "static" means don't export the symbol... */
       myfunc2(void)
       {
           myfunc3();
       }

       void
       myfunc(int ncalls)
       {
           if (ncalls > 1)
               myfunc(ncalls - 1);
           else
               myfunc2();
       }

       int
       main(int argc, char *argv[])
       {
           if (argc != 2) {
               fprintf(stderr, "%s num-calls\n", argv[0]);
               exit(EXIT_FAILURE);
           }

           myfunc(atoi(argv[1]));
           exit(EXIT_SUCCESS);
       }

SEE ALSO
       gcc(1), ld(1), dlopen(3), malloc(3)

COLOPHON
       This  page  is  part  of  release  3.22 of the Linux man-pages project.  A description of the project, and information about reporting bugs, can be found at http://www.ker-
       nel.org/doc/man-pages/.

GNU                               2008-06-14                      BACKTRACE(3)
```