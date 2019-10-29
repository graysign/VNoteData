# 获取ip  

```c++
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netdb.h>
#include <ifaddrs.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> 
/* some defines collected around the place: */

#define _PATH_PROCNET_IFINET6   "/proc/net/if_inet6"
#define IPV6_ADDR_LOOPBACK      0x0010U
#define IPV6_ADDR_LINKLOCAL     0x0020U
#define IPV6_ADDR_SITELOCAL     0x0040U
#define IPV6_ADDR_COMPATv4      0x0080U
 
int
main(int argc, char *argv[])
{
   int scope; /* among other stuff */

	/* looks like here we parse the contents of /proc/net/if_inet6: */
	 FILE * f = NULL;
	if ((f = fopen(_PATH_PROCNET_IFINET6, "r")) != NULL) {
		unsigned char buf[sizeof(struct in6_addr)] = {0}; 
		int if_idx,plen,scope,dad_status;
		char devname[20] = {0};
		char* scopestr = "Unknown"; 
		while (fscanf(f, "%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x %02x %02x %02x %02x %20s\n",
			  &buf[0],&buf[1],&buf[2],&buf[3],&buf[4],&buf[5],&buf[6],&buf[7],&buf[8],&buf[9],&buf[10],&buf[11],&buf[12],&buf[13],&buf[14],&buf[15],&if_idx, &plen, &scope, &dad_status, devname) != EOF) { 
			
			//printf("%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x \n", buf[0],buf[1],buf[2],buf[3],buf[4],buf[5],buf[6],buf[7],buf[8],buf[9],buf[10],buf[11],buf[12],buf[13],buf[14],buf[15]); 
			/* slightly later: */ 
			switch (scope) {
				case 0:
					scopestr = "Global";
					break;
				case IPV6_ADDR_LINKLOCAL:
					scopestr = "Link";
					break;
				case IPV6_ADDR_SITELOCAL:
					scopestr = "Site";
					break;
				case IPV6_ADDR_COMPATv4:
					scopestr = "Compat";
					break;
				case IPV6_ADDR_LOOPBACK:
					scopestr = "Host";
					break;
				default:
					scopestr = "Unknown";
			}  
			
			char str[INET6_ADDRSTRLEN]; 	
		    inet_ntop(AF_INET6, buf, str, INET6_ADDRSTRLEN);
			printf("%s\n", str);
			
	    }
		fclose(f);
	}  
   return 0;
}
```


# 获取网关  

```c++
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netdb.h>
#include <ifaddrs.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> 
#include <strings.h> 
#include <string.h>  
bool GetLocalIPv6Gateway(char const * devName, const char * golbaladdr)
{
    const char * PROCNET_ROUTE_IPV6_PATH   =   "/proc/net/ipv6_route";
    FILE * f = fopen(PROCNET_ROUTE_IPV6_PATH, "r");
	if (f == NULL) {
		return 0;
	}  
	unsigned char ipv6_buf[8][4+1]; 
    bzero(ipv6_buf, sizeof(ipv6_buf));

	unsigned char ipv6_prefix_buf[2+1];
    bzero(ipv6_prefix_buf, sizeof(ipv6_prefix_buf));

	unsigned char ipv6_main_route[32+1];
    bzero(ipv6_main_route, sizeof(ipv6_main_route));

	unsigned char ipv6_main_route_prefix_buf[2+1];
    bzero(ipv6_main_route_prefix_buf, sizeof(ipv6_main_route_prefix_buf));

	unsigned char ipv6_gateway_buf[8][5]; 
    bzero(ipv6_gateway_buf, sizeof(ipv6_gateway_buf));

	unsigned char ipv6_route_select_buf[8+1];//rt->rt6i_metric  rt6_select时使用，路由选择的条件。
    bzero(ipv6_route_select_buf, sizeof(ipv6_route_select_buf));

	unsigned char ipv6_route_refcnt_buf[8+1];//rt->dst.__refcnt 路由表管理时使用
    bzero(ipv6_route_refcnt_buf, sizeof(ipv6_route_refcnt_buf));

	unsigned char ipv6_route_use_buf[8+1];//rt->dst.__use 路由被使用次数。
    bzero(ipv6_route_use_buf, sizeof(ipv6_route_use_buf));

	unsigned char ipv6_route_flg_buf[8+1];//rt->rt6i_flags 是否刷新等。	
    bzero(ipv6_route_flg_buf, sizeof(ipv6_route_flg_buf));

	char devname_buf[20]; 
    bzero(devname_buf, sizeof(devname_buf));
    while (fscanf(f, "%4s%4s%4s%4s%4s%4s%4s%4s %s %s %s %4s%4s%4s%4s%4s%4s%4s%4s %s %s %s %s %20s\n",
		  ipv6_buf[0], ipv6_buf[1], ipv6_buf[2], ipv6_buf[3],ipv6_buf[4], ipv6_buf[5], ipv6_buf[6], ipv6_buf[7],
		  ipv6_prefix_buf, 
		  ipv6_main_route,
		  ipv6_main_route_prefix_buf,
		  ipv6_gateway_buf[0], ipv6_gateway_buf[1], ipv6_gateway_buf[2], ipv6_gateway_buf[3],ipv6_gateway_buf[4], ipv6_gateway_buf[5], ipv6_gateway_buf[6], ipv6_gateway_buf[7],
		  ipv6_route_select_buf,
		  ipv6_route_refcnt_buf,
		  ipv6_route_use_buf,
		  ipv6_route_flg_buf,
		  devname_buf) != EOF) {  
        int     route_flag  =   atoi((const char *)ipv6_route_flg_buf);
        bzero(ipv6_route_flg_buf, sizeof(ipv6_route_flg_buf));
        //if(!(route_flag & RTF_GATEWAY))
		printf("flag %d  \n", route_flag);
		if(!(route_flag & 0x0002))
        {
            continue;
        } 
		printf("route_flag_gateway %d  \n", (route_flag & 0x0002));
		
        if(strcmp(devname_buf, devName) != 0)
        {
            continue;
        }
		char addr6[46];
		char stripv6[64]; 
		struct sockaddr_in6 sap; 
        bzero(addr6, sizeof(addr6));
        bzero(stripv6, sizeof(stripv6));
        bzero(&sap, sizeof(struct sockaddr_in6));
		sprintf(addr6, "%s:%s:%s:%s:%s:%s:%s:%s",ipv6_buf[0], ipv6_buf[1], ipv6_buf[2], ipv6_buf[3],ipv6_buf[4], ipv6_buf[5], ipv6_buf[6], ipv6_buf[7]);		
	    inet_pton(AF_INET6, addr6, &sap.sin6_addr);
		inet_ntop(AF_INET6, &sap.sin6_addr, stripv6, sizeof(stripv6));
		printf("gateway:%s \n",stripv6);
        if(strcmp(stripv6, golbaladdr) != 0 && strcmp(stripv6, "::") != 0 )
        {
            continue;
        }

		bzero(addr6, sizeof(addr6));
        bzero(stripv6, sizeof(stripv6));
        bzero(&sap, sizeof(struct sockaddr_in6));
		sprintf(addr6, "%s:%s:%s:%s:%s:%s:%s:%s",ipv6_gateway_buf[0], ipv6_gateway_buf[1], ipv6_gateway_buf[2], ipv6_gateway_buf[3],ipv6_gateway_buf[4], ipv6_gateway_buf[5], ipv6_gateway_buf[6], ipv6_gateway_buf[7]);		
	    inet_pton(AF_INET6, addr6, &sap.sin6_addr);
		inet_ntop(AF_INET6, &sap.sin6_addr, stripv6, sizeof(stripv6));
		 
		printf("%s \n",stripv6);
		break;
	}
	
	fclose(f);
    return true; 			
}

 
int
main(int argc, char *argv[])
{
   GetLocalIPv6Gateway("eth0", "");
   return 0;
}
```