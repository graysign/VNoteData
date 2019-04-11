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