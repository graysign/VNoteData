# 环境

Windows7
Python3.65
scrapy1.74
PyInstaller3.5  

# 创建打包脚本

在与scrapy.cfg同路径创建start.py

```python
# -*- coding: utf-8 -*-from scrapy.crawler import CrawlerProcessfrom scrapy.utils.project import get_project_settings
 
# 必加的依赖
import scrapy.spiderloaderimport scrapy.statscollectorsimport scrapy.logformatterimport scrapy.dupefiltersimport scrapy.squeues import scrapy.extensions.spiderstateimport scrapy.extensions.corestatsimport scrapy.extensions.telnetimport scrapy.extensions.logstatsimport scrapy.extensions.memusageimport scrapy.extensions.memdebugimport scrapy.extensions.feedexportimport scrapy.extensions.closespiderimport scrapy.extensions.debugimport scrapy.extensions.httpcacheimport scrapy.extensions.statsmailerimport scrapy.extensions.throttle import scrapy.core.schedulerimport scrapy.core.engineimport scrapy.core.scraperimport scrapy.core.spidermwimport scrapy.core.downloader import scrapy.downloadermiddlewares.statsimport scrapy.downloadermiddlewares.httpcacheimport scrapy.downloadermiddlewares.cookiesimport scrapy.downloadermiddlewares.useragentimport scrapy.downloadermiddlewares.httpproxyimport scrapy.downloadermiddlewares.ajaxcrawlimport scrapy.downloadermiddlewares.chunkedimport scrapy.downloadermiddlewares.decompressionimport scrapy.downloadermiddlewares.defaultheadersimport scrapy.downloadermiddlewares.downloadtimeoutimport scrapy.downloadermiddlewares.httpauthimport scrapy.downloadermiddlewares.httpcompressionimport scrapy.downloadermiddlewares.redirectimport scrapy.downloadermiddlewares.retryimport scrapy.downloadermiddlewares.robotstxt import scrapy.spidermiddlewares.depthimport scrapy.spidermiddlewares.httperrorimport scrapy.spidermiddlewares.offsiteimport scrapy.spidermiddlewares.refererimport scrapy.spidermiddlewares.urllength import scrapy.pipelines import scrapy.core.downloader.handlers.httpimport scrapy.core.downloader.contextfactory

import scrapy.core.downloader.handlers.fileimport scrapy.core.downloader.handlers.ftpimport scrapy.core.downloader.handlers.datauriimport scrapy.core.downloader.handlers.s3

process = CrawlerProcess(get_project_settings())
# 参数1：爬虫名 参数2：域名
process.crawl('biqubao_spider',domain='biqubao.com')process.start()  # the script will block here until the crawling is finishedinput()
```

# 打包
  
 
`pyinstaller start.py`

  
打包完成后生成了三个文件：dist，build(可删)，start.spec（可删）  
  
问题1：  
关于导入robotparser库这个问题，本文参考的文章是导入了这个库，可是经过尝试后发现没有安装这个库。  
Scrapy会自动解析机器人协议的，所以尝试不导入这个库发现可以成功打包。  
  
问题2：  
可能你的Scrapy项目有使用其他的库，所以在这个打包脚本上你需要导入你使用的所有库。  
  
问题3：  
打包成功后运行不了程序，因为少两个问题，如下图
![](_v_images/20210120115301762_29838.png)

你需要在dist中添加scrapy文件夹，里面包含这两个文件，这两个文件在安装的scrapy的库中。  
  
  
# 打包单文件  
  
**打包单文件需要修改spec文件的datas数据,添加pyinstaller不能识别的文件  
其中('.','.')如果不增加的话，在移动可执行文件时就会报错,这个其实就是将当前目录放到打包后的根路径下(整个爬虫文件打包进去，使得可执行文件更大)

```python
# -*- mode: python ; coding: utf-8 -*-
block_cipher = None
a = Analysis(['start.py'],
             pathex=['G:\\1\\spider\\BookSpider'],
             binaries=[],
             datas=[('G:\\1\\spider\\BookSpider\\VERSION','scrapy'),('G:\\1\\spider\\BookSpider\\mime.types','scrapy'),('.','.')],
             hiddenimports=[],
             hookspath=[],
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
pyz = PYZ(a.pure, a.zipped_data,
             cipher=block_cipher)
exe = EXE(pyz,
          a.scripts,
          a.binaries,
          a.zipfiles,
          a.datas,
          [],
          name='start',
          debug=False,
          bootloader_ignore_signals=False,
          strip=False,
          upx=True,
          upx_exclude=[],
          runtime_tmpdir=None,
          console=True )
```

删除生成的dict和build文件后运行

`pyinstaller -F start.py start.exe`

  发现此时没有报错了(如果不修改spec文件打包单文件会报错)

发现了另外一个问题，当文件移动时还是会报错

其实已经打包完成了，只需要把可执行文件与scrapy.cfg放一起就行
