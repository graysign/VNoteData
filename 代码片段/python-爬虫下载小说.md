
```python
#!/usr/bin/python
#coding=utf-8
import urllib2
import urllib
import json
import hashlib
import time
class Book:
    __id            =   ""
    __source_id     =   ""
    __chapter       =   []
    __book_name     =   ""
    def search(self, name, author = ""):
        bFind   =   True
        self.__book_name    =   name
        url     =   "http://api.zhuishushenqi.com/book/fuzzy-search?" + urllib.urlencode({"query": name.encode("utf8")})
        req     =   urllib2.Request(url)
        req.add_header("Host", "api.zhuishushenqi.com")
        req.add_header("User-Agent", "ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
        response =  urllib2.urlopen(req)
        content  =  response.read()
        books    =  json.loads(content)["books"]
        for i in range(len(books)):
            book = books[i]
            info = str(i+1)+". 小说名【"+book["title"].encode("utf8")+"】 作者【"+book["author"].encode("utf8")+"】"
            print info
        index = int(input("请选择小说:"))
        self.__id = books[index - 1]["_id"]
        return True

    def source(self):
        bFind       =   False
        strUrl      =   "http://api.zhuishushenqi.com/toc?view=summary&book="+self.__id
        req         =    urllib2.Request(strUrl)
        req.add_header("Host", "api.zhuishushenqi.com")
        req.add_header("User-Agent", "ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
        response    =   urllib2.urlopen(req)
        content     =   response.read()
        source      =   json.loads(content)

        for i in range(len(source)):
            print str(i+1)+": "+source[i]["name"]
        index   =       int(input( "请选择下载源:"))
        self.__source_id = source[index -1 ]["_id"]
        return True

    def chapter(self):
        bFind       =   False
        strUrl      =   "http://api.zhuishushenqi.com/toc/"+self.__source_id+"?view=chapters"
        req = urllib2.Request(strUrl)
        req.add_header("Host", "api.zhuishushenqi.com")
        req.add_header("User-Agent", "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36")
        response            =   urllib2.urlopen(req)
        content             =   response.read()
        self.__chapter      =   json.loads(content)["chapters"]
        if len(self.__chapter) > 0 :
            bFind           =   True
        return bFind

    def chapter_content(self):
        file  = open(self.__book_name+'.txt', 'a+')
        count = len(self.__chapter)
        for i in range(len(self.__chapter)):
            chapter     =   self.__chapter[i]
            timp        =   str(int(time.time()))
            k           =   hashlib.md5(timp).hexdigest()[8:-8]
            link        =   urllib.quote(chapter["link"])
            strUrl      =   "http://chapterup.zhuishushenqi.com/chapter/"+link+"?k="+k+"&t="+timp
            req = urllib2.Request(strUrl)
            req.add_header("Host", "chapter2.zhuishushenqi.com")
            req.add_header("User-Agent","ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
            req.add_header("X-User-Agent", "ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
            req.add_header("X-Device-Id", "372595914172b477")
            print "正在下载【" + chapter["title"].encode("utf8")+"】URL【"+strUrl+"】"
            try:
                response = urllib2.urlopen(req)
            except  BaseException:
                continue
            else:
                content  = json.loads(response.read())
                if content["ok"]    ==  True :
                    title       =       content["chapter"]["title"] != chapter["title"] and  chapter["title"] or content["chapter"]["title"]
                    cpContent   =       content["chapter"].has_key("cpContent") and content["chapter"]["cpContent"] or ""
                    cpContent   +=      content["chapter"].has_key("body") and content["chapter"]["body"] or ""

                file.writelines([title.encode("utf8"), cpContent.encode("utf8")])

        file.close()
        return  None

    def select_chapter(self):
        for i in range(len(self.__chapter)):
            chapter = self.__chapter[i]
            print str(i)+":" + chapter["title"].encode("utf8") + "】"
        index =     int(input("请选择章节:")) - 1
        while index > -1 :
            chapter = self.__chapter[index]
            timp = str(int(time.time()))
            k = hashlib.md5(timp).hexdigest()[8:-8]
            link = urllib.quote(chapter["link"])
            strUrl = "http://chapterup.zhuishushenqi.com/chapter/" + link + "?k=" + k + "&t=" + timp
            req = urllib2.Request(strUrl)
            req.add_header("Host", "chapter2.zhuishushenqi.com")
            req.add_header("User-Agent", "ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
            req.add_header("X-User-Agent", "ZhuiShuShenQi/3.135.3 (Android 4.4.2; XiaoMi MI 5 / Tencent MI 5; )")
            req.add_header("X-Device-Id", "372595914172b477")
            try:
                response = urllib2.urlopen(req)
            except  BaseException:
                continue
            else:
                content  = json.loads(response.read())
                if content["ok"]    ==  True :
                    title       =       content["chapter"]["title"] != chapter["title"] and  chapter["title"] or content["chapter"]["title"]
                    cpContent   =       content["chapter"].has_key("cpContent") and content["chapter"]["cpContent"] or ""
                    cpContent   +=      content["chapter"].has_key("body") and content["chapter"]["body"] or ""
                    print cpContent.encode("utf8")
                    next_chapter    =   int(input("是否继续(0继续|1退出):"))
                    if next_chapter == 0 :
                        index = index + 1
                        continue
                    else:
                        break
        return None

type = int(input("请选择小说模式(0下载|1阅读):"))

book  =    Book()
if book.search(u"弄潮","") :
   if book.source():
      if book.chapter() :
          if type == 0 :
              book.chapter_content()
          else:
              book.select_chapter()
```




