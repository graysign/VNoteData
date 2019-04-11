```php
<?php
class FSSH{
	private $conn;
	private $shell;

	/**
	* key=String 密码认证,key=array('pub'=>,'pri'=>,'type'=>,'phrase'=>)密钥认证
	* 密钥认证type分为两种：ssh-rsa,ssh-dss 
	* $host[addr]=String 地址,$host['fp']=array() 服务器指纹
	*/
	function __construct($host,$user,$key){
		if(empty($host['addr'])){
			debug_print('Host cant\'t be empty',E_USER_ERROR);
		}
		if(empty($host['fp'])){
			debug_print('finger print is not specified',E_USER_ERROR);
		}
		$this->stdin=fopen('php://stdin','r');
		$this->stdout=fopen('php://stdout','w');
		if(false!==strpos($host['addr'],':')){
			$temp=explode(':',$host['addr']);
			$host['addr']=$temp[0];
			$port=$temp[1];
		}else{
			$port=22;
		}
		if(is_string($key) || empty($key['type'])){
			$methods=null;
		}else{
			$methods=array('hostkey'=>$key['type']);
		}
		$conn=ssh2_connect($host['addr'],$port,$methods,array('disconnect'=>array($this,'disconnect')));
		$fp=ssh2_fingerprint($conn,SSH2_FINGERPRINT_MD5);
		$success=false;
		$fpOK=false;
		if(in_array($fp,$host['fp'])){
			$fpOK=true;
		}else{
			fwrite($this->stdout,"$fp\nIs fingerprint OK ?(y/n)");
			$input=strtolower(stream_get_line($this->stdin,1));
			if($input=='y'){
				$fpOK=true;
			}else{
				$fpOK=false;
			}
		}
		if($fpOK){
			if(is_array($key)){
				if (ssh2_auth_pubkey_file($conn,$user,$key['pub'],$key['pri'],$key['phrase'])){
					$success=true;
				}else{
					debug_print('Public Key Authentication Failed',E_USER_ERROR);
				}
			}elseif(is_string($key)){
				if(ssh2_auth_password($conn,$user,$key)){
					$success=true;
				}else{
					debug_print('Password Authentication Failed',E_USER_ERROR);
				}
			}
		}else{
			debug_print('Fingerprint is invalid',E_USER_ERROR);
		}
		if($success){
			$this->conn=$conn;
			$this->shell=ssh2_shell($conn,null,null,1024);
		}
		return $success;
	}

	function shell(){
		//最后一条命令
		$last='';
		//先结束shell，再结束while
		$signalTerminate=false;
		while(true){
			$cmd=$this->fread($this->stdin);
			$out=stream_get_contents($this->shell,1024);
			if(!empty($out) and !empty($last)){
				$l1=strlen($out);
				$l2=strlen($last);
				$l=$l1>$l2?$l2:$l1;
				$last=substr($last,$l);
				$out=substr($out,$l);
			}
			echo ltrim($out);
			if($signalTerminate){
				break;
			}
			if(in_array(trim($cmd),array('exit'))){
				$signalTerminate=true;
			}
			if(!empty($cmd)){
				$last=$cmd;
				fwrite($this->shell,$cmd);
			}
		}
	}

	//解决windows命令行的读取问题，没有别的办法了。
	private function fread($fd){
		static $data='';
		$read = array($fd);
		$write = array();
		$except = array();
		$result = stream_select($read,$write,$except,0,1000);
		if($result === false)
			debug_print('stream_select failed',E_USER_ERROR);
		if($result !== 0){
			$c= stream_get_line($fd,1);
			if($c!=chr(13))
				$data.=$c;
			if($c==chr(10)){
				$t=$data;
				$data='';
				return $t;
			}
		}
	}

	function __destruct(){
		fclose($this->stdin);
		fclose($this->stdout);
		$this->disconnect();
	}

	private function disconnect(){
		if(is_resource($this->conn)){
			unset($this->conn);
			fclose($this->shell);
		}
	}
}
$ssh=new FSSH(array('addr'=>'127.0.0.1'),'root','password');
$ssh->shell();

```