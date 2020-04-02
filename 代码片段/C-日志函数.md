# 一. 打印日志
```c
void  LOG(const char *format, ...)
{
	va_list argp;
	char buf[1024]; 
	FILE *fp = NULL;
	
	if( (fp = fopen("EnableLog" ,"r")) == NULL){
		return;
	}else{
		fclose(fp);
		fp = NULL;
	}
		
	va_start(argp, format);
	#ifdef WIN32
	_vsnprintf(buf, sizeof(buf), format, argp);
	#else
	vsnprintf(buf, sizeof(buf), format, argp);
	#endif
	va_end(argp);

	
	if((fp = fopen("log.txt" ,"a+")) != NULL)
	{
		#ifdef WIN32
			SYSTEMTIME stLocal;
			::GetLocalTime(&stLocal);
			fprintf(fp, "[DEBUG %2.u:%2.u:%2.u %u] ", stLocal.wHour, stLocal.wMinute, stLocal.wSecond, stLocal.wMilliseconds);
		#else	
			time_t now;     
			struct tm  *timenow;
			time(&now); 
			timenow = localtime(&now); 
			fprintf(fp, "[DEBUG %2.u:%2.u:%2.u] ", timenow->tm_hour, timenow->tm_min, timenow->tm_sec);
		#endif
		fwrite(buf,strlen(buf),sizeof(char),fp);
		fflush(fp);
		fclose(fp);
	}
	 
}
```


# 二. 打印16进制数据

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <ctype.h>

void PrintHex(char* fragment, int length)
{
    if(fragment == NULL || length == 0)
        return;
    const int octest_per_line = 16;
    char buf[1024];
    char charbuf[1024];
    buf[0]  =   '\0';
    char *p =   buf;
    char *d =   charbuf;
    int this_line_begins_at =   0;
    int pos;
    for(pos = 0; pos < length;){
        char c  =   fragment[pos];
        sprintf(p, "%02x ",c);
        p   =   strchr(p, '\0');
        if(isprint(c))
            *d++=c;
        else
            *d++='.';
        ++pos;
        if(pos - this_line_begins_at == octest_per_line){
            *d  =   '\0';
            printf("%s  %s \n", buf, charbuf);   
            buf[0]  =   '\0';
            charbuf[0] =   '\0';
            p   =   buf;
            d   =   charbuf;
            this_line_begins_at =   pos;
        }    
        
    } 
    if(pos - this_line_begins_at > 0){
        *d  =   '\0';
        printf("%-*.*s  %s \n", octest_per_line * 3, octest_per_line * 3, buf, charbuf);
    }
        
        
    
}

int
main(int argc, char** argv)
{
    char* buf = "1234567890abcdefghigklmnopqrstuvwxyz"; 
    PrintHex(buf, 36); 


    return 0;
}
```