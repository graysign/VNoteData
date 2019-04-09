```python
#!/usr/bin/python
#coding=utf-8
import urllib2

file = open("C:\\Users\\dell\\Downloads\\5opdCm2T.m3u8")
code = open("C:\\Users\\dell\\Downloads\\流浪地球.ts", "wb+")
while True:
    line = file.readline()
    if not line:
        break
    if line[0] == '#':
        continue
    print line
    f = urllib2.urlopen("https://vip2.pp63.org"+line)
    data = f.read()
    code.write(data)
file.close()
code.close()
```