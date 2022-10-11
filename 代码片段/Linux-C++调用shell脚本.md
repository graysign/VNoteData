# Linux-C++调用shell脚本

```c++
#include <stdio.h>
int main(int argc,char** argv)
{
    FILE *fp = NULL;
    char buf[100]={0};
	char command[1024] = {0};
	if(argc > 1 ){
		sprintf(command, "sh /temp/test.sh %s",argv[1]);
	}else{
		sprintf(command, "sh /temp/test.sh %s",argv[0]);
	}
    fp = popen(command, "r");
    if(fp) {
        int ret =  fread(buf,1,sizeof(buf)-1,fp);
        if(ret > 0) {
            printf("%s",buf);
        }
        pclose(fp);
    }
	return 0;
}
```