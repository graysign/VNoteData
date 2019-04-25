**以下代码在 python 3.5 + jupyter notebook 中运行测试无误！**  

```python
# 我们爬取网页的目的，无非是先定位到DOM树的节点，然后取其文本或属性值

myPage = '''<html>
        <title>TITLE</title>
        <body>
        <h1>我的博客</h1>
        <div>我的文章</div>
        <div id="photos">
         <img src="pic1.jpeg"/><span id="pic1">PIC1 is beautiful!</span>
         <img src="pic2.jpeg"/><span id="pic2">PIC2 is beautiful!</span>
         <p><a href="http://www.example.com/more_pic.html">更多美图</a></p>
         <a href="http://www.baidu.com">去往百度</a>
         <a href="http://www.163.com">去往网易</a>
         <a href="http://www.sohu.com">去往搜狐</a>
        </div>
        <p class="myclassname">Hello,\nworld!<br/>-- by Adam</p>
        <div class="foot">放在尾部的其他一些说明</div>
        </body>
        </html>'''
        
html = etree.fromstring(myPage)

# 一、定位
divs1 = html.xpath('//div')
divs2 = html.xpath('//div[@id]')
divs3 = html.xpath('//div[@class="foot"]')
divs4 = html.xpath('//div[@*]')
divs5 = html.xpath('//div[1]')
divs6 = html.xpath('//div[last()-1]')
divs7 = html.xpath('//div[position()<3]')
divs8 = html.xpath('//div|//h1')
divs9 = html.xpath('//div[not(@*)]')

# 二、取文本 text() 区别 html.xpath('string()')
text1 = html.xpath('//div/text()')
text2 = html.xpath('//div[@id]/text()')
text3 = html.xpath('//div[@class="foot"]/text()')
text4 = html.xpath('//div[@*]/text()')
text5 = html.xpath('//div[1]/text()')
text6 = html.xpath('//div[last()-1]/text()')
text7 = html.xpath('//div[position()<3]/text()')
text8 = html.xpath('//div/text()|//h1/text()')


# 三、取属性 @
value1 = html.xpath('//a/@href')
value2 = html.xpath('//img/@src')
value3 = html.xpath('//div[2]/span/@id')


# 四、定位（进阶）
# 1.文档(DOM)元素(Element)的find，findall方法
divs = html.xpath('//div[position()<3]')
for div in divs:
    ass = div.findall('a')  # 这里只能找到:div->a, 找不到:div->p->a
    for a in ass:
        if a is not None:
            #print(dir(a))
            print(a.text, a.attrib.get('href')) #文档(DOM)元素(Element)的属性：text, attrib

# 2.与1等价
a_href = html.xpath('//div[position()<3]/a/@href')
print(a_href)

# 3.注意与1、2的区别
a_href = html.xpath('//div[position()<3]//a/@href')
print(a_href)  

```