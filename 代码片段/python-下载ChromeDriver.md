```python
#!/usr/bin/python
#coding=utf-8
#http://npm.taobao.org/mirrors/chromedriver/
import urllib
import os

def Schedule(a,b,c):
    '''''
    a:已经下载的数据块
    b:数据块的大小
    c:远程文件的大小
   '''
    per = 100.0 * a * b / c
    if per > 100 :
        per = 100
    print '%.2f%%' % per

DownLoadUrl =   "http://chromedriver.storage.googleapis.com/"
ChromeDriverMapping =   {
    "2.29":range(56,58+1),
    "2.28":range(55,57+1),
    "2.27":range(54,56+1),
    "2.26":range(53,55+1),
    "2.25":range(53,55+1),
    "2.24":range(52,57+1),
    "2.23":range(51,53+1),
    "2.22":range(49,52+1),
    "2.21":range(46,50+1),
    "2.20":range(43,48+1),
    "2.19":range(43,47+1),
    "2.18":range(43,46+1),
    "2.17":range(42,43+1),
    "2.13":range(42,45+1),
    "2.15":range(40,43+1),
    "2.14":range(39,42+1),
    "2.13":range(38,41+1),
    "2.12":range(36,40+1),
    "2.11":range(36,40+1),
    "2.10":range(33,36+1),
    "2.9": range(31,34+1),
    "2.8": range(30,33+1),
    "2.7": range(30,33+1),
    "2.6": range(29,32+1),
    "2.5": range(29,32+1),
    "2.4": range(29,32+1)
}
ChromeDriverList    =   []
ChromeVersion = int(raw_input("Enter chrome version(eg: 51 ): "));
for (drivers,versions) in ChromeDriverMapping.items():
    for i in range(len(versions)) :
        if ChromeVersion == versions[i]:
            ChromeDriverList.append(drivers)
            break

for i in range(len(ChromeDriverList)):
    print str(i+1)+"  ChromeDriverVersion:("+ str(ChromeDriverList[i])+")"

if len(ChromeDriverList) == 0 :
    exit()

ChromeDriverVersion= int(raw_input("Enter number: "));
if ChromeDriverVersion > len(ChromeDriverList):
    exit()
FileName    =   "chromedriver_win32.zip"
DownLoadUrl = DownLoadUrl+ChromeDriverList[ChromeDriverVersion - 1]+"/"+FileName
local = os.path.join("./",FileName)
urllib.urlretrieve(DownLoadUrl,local,Schedule)
```