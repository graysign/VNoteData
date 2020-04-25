```php
<?php
class	Article{ 
	private $catalog_url				=	"";
	private	$title_chapter_rule			=	[];
	private	$title_href_rule			=	[];
	private $content_rule				=	[];
	private $chapter					=	[];
	
	function __construct(){
		$this->chapter["title"] 				= 	[];
		$this->chapter["href"] 					= 	[];
		$this->chapter["content"] 				= 	[];
	}
	public function AddTitleChapterRule($rule){array_push($this->title_chapter_rule, $rule);}
	public function AddTitleHerfRule($rule){array_push($this->title_href_rule, $rule);}
	public function AddContentRule($rule){array_push($this->content_rule, $rule);}
	public function SetCatalogUrl($url){$this->catalog_url = $url;}
	private function GetHtml($url){ 
		$ch 	   = curl_init();
		$headers   = array();
		$headers[] = 'Host: www.hujuge.com';
		$headers[] = 'Cache-Control: no-cache';
		$headers[] = 'Content-Type: application/x-www-form-urlencoded; charset=utf-8';
		$headers[] = 'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:28.0) Gecko/20100101 Firefox/28.0';
		$headers[] = 'Accept: */*';
		
		curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
		curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
		curl_setopt($ch, CURLOPT_URL, $url);
		$html = curl_exec($ch); 
		curl_close($ch);
		return $html;
		
	}
	private function GetContent($html, $reg)
	{
		$dom 	=  new DOMDocument();
		@$dom->loadHTML($html);
		$xPath = new DOMXPath($dom);
		return @$xPath->query($reg);
	}
	
	private function GetChapter(){
		$html 			=	$this->GetHtml($this->catalog_url);
		foreach($this->title_chapter_rule as $rule){ 
			$elements	=	$this->GetContent($html, $rule);
			foreach($elements as $element){
				array_push($this->chapter["title"], mb_convert_encoding($element->nodeValue, "GBK", "UTF-8"));
			}
		}
		foreach($this->title_href_rule as $rule){
			$elements	=	$this->GetContent($html, $rule);
			foreach($elements as $element){
				array_push($this->chapter["href"], $element->nodeValue);
			}
		}
	}
	public function PrintChapter(){
		for($i = 0 ; $i < count($this->chapter["title"]) ; $i++){
			var_dump($this->chapter["href"][$i]."|".$this->chapter["title"][$i]);
		}
		
	}
	public function Run(){
		$this->GetChapter(); 
	}
}
	/*$url 		= "http://www.hujuge.com/390218/index.html"; 
	$elements	=  GetContent(GetHtml($url),"/html/body/section[4]/dl/dd/a/@href");
	foreach ($elements as $e) {
		$content_url		=	"http://www.hujuge.com".$e->nodeValue;
		$content_html		=	GetHtml($content_url);
		$name			=	GetContent($content_html,'/html/body/section[3]/div[1]/div[1]/text()[4]');
		$content		=	GetContent($content_html,'//*[@id="chapterContent"]');
		foreach($name as $n){
			file_put_contents("log.txt",$n->nodeValue.PHP_EOL, FILE_APPEND);
		}	
		foreach($content as $n){
			file_put_contents("log.txt",$n->nodeValue.PHP_EOL, FILE_APPEND);
		}
	}*/
	/*$demo 	=	new Article();
	$demo->SetCatalogUrl("http://www.hujuge.com/390218/index.html");
	$demo->AddTitleChapterRule('/html/body/section[4]/dl/dd/a/@title');
	$demo->AddTitleHerfRule("/html/body/section[4]/dl/dd/a/@href");
	$demo->Run();
	$demo->PrintChapter();
	*/
	///////////////////////////////////////////////////////////////
	function GetHtml($form){ 
		$url	   = $form['action'];	
		$ch 	   = curl_init();
		$headers   = array();
		$headers[] = 'Cache-Control: no-cache';
		$headers[] = 'Content-Type: application/x-www-form-urlencoded; charset=utf-8';
		$headers[] = 'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:28.0) Gecko/20100101 Firefox/28.0';
		$headers[] = 'Accept: */*';
		foreach($form['headers'] as $d){
			array_push($headers,$d);
		}
		if($form['method'] == "post"){
			curl_setopt($ch, CURLOPT_POST, true);
			curl_setopt($ch, CURLOPT_POSTFIELDS, $form['date']);
		}else{
			$formString=	"";
			foreach($form['date'] as $k => $v){
				$formString = $k."=".urlencode($v)."&";
			}
			$url	=	$url ."?".$formString;
		}
		if($form['https']){
			curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); //不验证证书
			curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false); //不验证证书
		}
		curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
		curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
		curl_setopt($ch, CURLOPT_URL, $url);
		$html = curl_exec($ch); 
		curl_close($ch);
		return $html;
		
	}
	function GetContent($html, $reg)
	{
		$dom 	=  new DOMDocument();
		@$dom->loadHTML($html);
		$xPath = new DOMXPath($dom);
		return @$xPath->query($reg);
	}
	$form	=	[
		"action"	=>	"https://www.qidian.com/search",
		"method"	=>	"get",
		"date"		=>	["kw"	=> "重生完美时代"],
		"headers"	=>	["Host: www.qidian.com", "Referer:https://www.qidian.com/"],
		"https"		=>	true,
		"rule"		=>	[
			"href"	=>	'//*[@id="result-list"]/div/ul/li[1]/div[2]/h4/a/@href',
			"name"	=>	'//*[@id="result-list"]/div/ul/li[1]/div[2]/h4/a/cite/text()',
			"author"=>	'//*[@id="result-list"]/div/ul/li[1]/div[2]/p[1]/a[1]/text()',
		]
	];
	$html		=	GetHtml($form);
	$href		=	GetContent($html, $form["rule"]["href"])[0]->nodeValue;
	$name		=	GetContent($html, $form["rule"]["name"])[0]->nodeValue;
	$author		=	GetContent($html, $form["rule"]["author"])[0]->nodeValue;
	
	var_dump($href);
	var_dump(mb_convert_encoding($name, "GBK", "UTF-8"));
	var_dump(mb_convert_encoding($author, "GBK", "UTF-8"));
?>

```