GitHub地址    https://github.com/CrawlScript/WebCollector    

Using Maven  
```xml
<dependency>
    <groupId>cn.edu.hfut.dmic.webcollector</groupId>
    <artifactId>WebCollector</artifactId>
    <version>2.73-alpha</version>
</dependency>
```
DEMO：  
用WebCollector制作一个爬取《知乎》并进行问题精准抽取的爬虫:
```java
public class ZhihuCrawler extends BreadthCrawler{

    /*visit函数定制访问每个页面时所需进行的操作*/
    @Override
    public void visit(Page page) {
        String question_regex="^http://www.zhihu.com/question/[0-9]+";
        if(Pattern.matches(question_regex, page.getUrl())){
            System.out.println("正在抽取"+page.getUrl());
            /*抽取标题*/
            String title=page.getDoc().title();
            System.out.println(title);
            /*抽取提问内容*/
            String question=page.getDoc().select("div[id=zh-question-detail]").text();
            System.out.println(question);

        }
    }

    /*启动爬虫*/
    public static void main(String[] args) throws IOException{  
        ZhihuCrawler crawler=new ZhihuCrawler();
        crawler.addSeed("http://www.zhihu.com/question/21003086");
        crawler.addRegex("http://www.zhihu.com/.*");
        crawler.start(5);  
    }


}
```