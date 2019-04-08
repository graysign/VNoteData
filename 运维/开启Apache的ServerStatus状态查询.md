# 开启Apache的ServerStatus状态查询
开启apache服务器中的Server Status状态查询。  
在Apache中，对于纷繁复杂的日志文件，如果靠分析日志或者查看服务器进程来监视Apache运行状态的话，比较繁冗。  
在Apache 1.3.2及以后的版本中，apache自带一个查看Apache状态的功能模块server-status。  
这里介绍下这个模块的开启办法，供大家参考。  
1. 打开Apache Server Status  
    如果Apache配置文件`httpd.conf`或`extra/httpd-info.conf`中有`LoadModule status_module modules/mod_status.so`话，说明Apache已经加载了此模块；  
    或编译的时候加上了`--enable-module=so`也表明服务器支持server-status。  
    如果Apache没有加载这个模块，如果是linux服务器，就得重新编译Apache，加上`--enable-module=so`参数即可；  
    如果windows系统的话，无需任何编译，只要把`LoadModule status_module modules/mod_status.so`这句加上，如果前面有带#号，开启的话，需要将#去除。  
```config
<location /jbxue-server-status>
  SetHandler server-status
  Order Deny,Allow
  Deny from all
  Allow from localhost
</location>
ExtendedStatus On
```
配置节说明：  
以上是一个完整的server-status的配置。  
第一行的jbxue-server-status表示以后可以用类似`http://localhost/jbxue-server-status`来访问，同时`http://localhost/jbxue-server-status?refresh=N`将表示访问  
状态页面可以每N秒自动刷新一次；  
    
```config
Deny from表示禁止的访问地址；
Allow from表示允许的地址访问；
ExtendedStatus On表示的是待会访问的时候能看到详细的请求信息。
```
    另外，该设置仅能用于全局设置，不能在特定的虚拟主机中打开或关闭。  
    启用扩展状态信息将会导致服务器运行效率降低。