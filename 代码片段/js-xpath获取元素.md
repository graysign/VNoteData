```javascript
//返回匹配到的第一个xpath对应的dom节点。
function x(xpath) {
  var result = document.evaluate(xpath, document, null, XPathResult.ANY_TYPE, null);
  return result.iterateNext()
}
function _x(STR_XPATH) {
    var xresult = document.evaluate(STR_XPATH, document, null, XPathResult.ANY_TYPE, null);
    var xnodes = [];
    var xres;
    while (xres = xresult.iterateNext()) {
        xnodes.push(xres);
    }
 
    return xnodes;
}

//返回匹配到xpath的dom节点个数。
function x(xpath) {
  var result = document.evaluate(xpath, document, null, XPathResult.ANY_TYPE, null);
  var i = 0;
  while(result.iterateNext()){
    i++;
  }
  return i;
}



```