可以用ajax get导出文件,从而最大程度上减少浏览器询问可能,也使下载更为灵活,目前新浪爱问(http://ishare.iask.sina.com.cn)就是这么做的
StartGETRequest方法,用于发起一个ajax get请求:  
```javascript
function StartGETRequest(url, handler)
{
    xmlhttp = null;
    if (is_ie) {    
        var control = (is_ie5) ? "Microsoft.XMLHTTP" : "Msxml2.XMLHTTP";
        try {      
            xmlhttp = new ActiveXObject(control);
        } catch(e) {
            alert("You need to enable active scripting and activeX controls");
            DumpException(e);
        }
    } else {
        xmlhttp = new XMLHttpRequest();
    }
    xmlhttp.onreadystatechange = function() {handler();}
    if (url.indexOf("?") != -1){
        var urltemp = url + "&rand=" + UniqueNum();
    } else {
        var urltemp = url + "?rand=" + UniqueNum();
    }
    //alert(urltemp);
    xmlhttp.open('GET', urltemp, true);
    
    xmlhttp.send(null); 
}
```
下载一个文件则使用:  
```javascript
StartGETRequest("http://abc.com/xxx.zip",function(){alert("下载完成")});
```
一次下载多个文件则可以:  
```javasctip
StartGETRequest("http://abc.com/1.zip",function(){});
StartGETRequest("http://abc.com/2.zip",function(){});
StartGETRequest("http://abc.com/3.zip",function(){});
```