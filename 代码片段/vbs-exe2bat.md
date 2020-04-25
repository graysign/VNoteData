```vbs
'exe2bat.vbs
fp=wscript.arguments(0) 
fn=right(fp,len(fp)-instrrev(fp,"")) 
with createobject("adodb.stream") 
.type=1:.open:.loadfromfile fp:str=.read:sl=lenb(str) 
end with 
sll=sl mod 65536:slh=sl65536 
with createobject("scripting.filesystemobject").opentextfile(fp&".bat",2,true) 
.write "@echo str=""" 
for i=1 to sl 
bt=ascb(midb(str,i,1)) 
if bt<16 then .write "0" 
.write hex(bt) 
if i mod 128=0 then .write """_>>debug.vbs"+vbcrlf+"@echo +""" 
next 
.writeline """>>debug.vbs"+vbcrlf+"@echo with wscript.stdout:r=vbcrlf"_ 
+":for i=1 to len(str) step 48:.write ""e""+hex(256+(i-1)/2)"_ 
+":for j=i to i+46 step 2:.write "" ""+mid(str,j,2):next:.write r:next>>debug.vbs" 
.writeline "@echo .write ""rbx""+r+"""+hex(slh)+"""+r+""rcx""+r+"""+hex(sll)_ 
+"""+r+""n debug.tmp""+r+""w""+r+""q""+r:end with"_ 
+">>debug.vbs&&cscript //nologo debug.vbs|debug.exe>nul&&ren debug.tmp """&fn&"""&del debug.vbs" 
end with 
```

`用法: cscript  exe2bat.vbs  putty.exe`