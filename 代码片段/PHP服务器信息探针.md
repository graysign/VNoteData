```php
<?
error_reporting(0); //抑制所有错误信息
@header("content-Type: text/html; charset=utf-8"); //语言强制
ob_start();
date_default_timezone_set('Asia/Shanghai');//此句用于消除时间差
$time_start = microtime_float();
 
/**
* 
*/
class ServerInfo{
    //服务器参数
    public $S = array(
        'YourIP', //你的IP
        'DomainIP', //服务器域名和IP及进程用户名
        'Flag', //服务器标识
        'OS', //服务器操作系统具体
        'Language', //服务器语言
        'Name', //服务器主机名
        'Email', //服务器管理员邮箱
        'WebEngine', //服务器WEB服务引擎
        'WebPort', //web服务端口
        'WebPath', //web路径
        'ProbePath', //本脚本所在路径
        'sTime' //服务器时间
        );
 
    public $sysInfo; //系统信息，windows和linux
    public $CPU_Use;
    public $hd = array(
        't', //硬盘总量
        'f', //可用
        'u', //已用
        'PCT', //使用率
        );
    public $NetWork = array(
        'NetWorkName', //网卡名称
        'NetOut', //出网总量
        'NetInput', //入网总量
        'OutSpeed', //出网速度
        'InputSpeed' //入网速度
        ); //网卡流量
 
    function __construct(){
        $this->S['YourIP'] = @$_SERVER['REMOTE_ADDR'];
        $domain = $this->OS()?$_SERVER['SERVER_ADDR']:@gethostbyname($_SERVER['SERVER_NAME']);
        $this->S['DomainIP'] = @get_current_user().' - '.$_SERVER['SERVER_NAME'].'('.$domain.')';
        $this->S['Flag'] = empty($this->sysInfo['win_n'])?@php_uname():$this->sysInfo['win_n'];
        $os = explode(" ", php_uname());
        $oskernel = $this->OS()?$os[2]:$os[1];
        $this->S['OS'] = $os[0].'内核版本：'.$oskernel;
        $this->S['Language'] = getenv("HTTP_ACCEPT_LANGUAGE");
        $this->S['Name'] = $this->OS()?$os[1]:$os[2];
        $this->S['Email'] = $_SERVER['SERVER_ADMIN'];
        $this->S['WebEngine'] = $_SERVER['SERVER_SOFTWARE'];
        $this->S['WebPort'] = $_SERVER['SERVER_PORT'];
        $this->S['WebPath'] = $_SERVER['DOCUMENT_ROOT']?str_replace('\\','/',$_SERVER['DOCUMENT_ROOT']):str_replace('\\','/',dirname(__FILE__));
        $this->S['ProbePath'] = str_replace('\\','/',__FILE__)?str_replace('\\','/',__FILE__):$_SERVER['SCRIPT_FILENAME'];
        $this->S['sTime'] = date('Y-m-d H:i:s');
 
        $this->sysInfo = $this->GetsysInfo();
        //var_dump($this->sysInfo);
 
        $CPU1 = $this->GetCPUUse();
        sleep(1);
        $CPU2 = $this->GetCPUUse();
        $data = $this->GetCPUPercent($CPU1, $CPU2);
        $this->CPU_Use =$data['cpu0']['user']."%us,  ".$data['cpu0']['sys']."%sy,  ".$data['cpu0']['nice']."%ni, ".$data['cpu0']['idle']."%id,  ".$data['cpu0']['iowait']."%wa,  ".$data['cpu0']['irq']."%irq,  ".$data['cpu0']['softirq']."%softirq";
        if(!$this->OS()) $this->CPU_Use = '目前只支持Linux系统';
 
        $this->hd = $this->GetDisk();
        $this->NetWork = $this->GetNetWork();
    }
    public function OS(){
        return DIRECTORY_SEPARATOR=='/'?true:false;
    }
    public function GetsysInfo(){
        switch (PHP_OS) {
            case 'Linux':
                $sysInfo = $this->sys_linux();
                break;
            case 'FreeBSD':
                $sysInfo = $this->sys_freebsd();
                break;
            default:
                # code...
                break;
        }
        return $sysInfo;
    }
    public function sys_linux(){ //linux系统探测
        $str = @file("/proc/cpuinfo"); //获取CPU信息
        if(!$str) return false;
        $str = implode("", $str);
        @preg_match_all("/model\s+name\s{0,}\:+\s{0,}([\w\s\)\(\@.-]+)([\r\n]+)/s", $str, $model); //CPU 名称
        @preg_match_all("/cpu\s+MHz\s{0,}\:+\s{0,}([\d\.]+)[\r\n]+/", $str, $mhz); //CPU频率
        @preg_match_all("/cache\s+size\s{0,}\:+\s{0,}([\d\.]+\s{0,}[A-Z]+[\r\n]+)/", $str, $cache); //CPU缓存
        @preg_match_all("/bogomips\s{0,}\:+\s{0,}([\d\.]+)[\r\n]+/", $str, $bogomips); //
        if(is_array($model[1])){
            $cpunum = count($model[1]);
            $x1 = $cpunum>1?' ×'.$cpunum:'';
            $mhz[1][0] = ' | 频率:'.$mhz[1][0];
            $cache[1][0] = ' | 二级缓存:'.$cache[1][0];
            $bogomips[1][0] = ' | Bogomips:'.$bogomips[1][0];
            $res['cpu']['num'] = $cpunum;
            $res['cpu']['model'][] = $model[1][0].$mhz[1][0].$cache[1][0].$bogomips[1][0].$x1;
            if(is_array($res['cpu']['model'])) $res['cpu']['model'] = implode("<br />", $res['cpu']['model']);
            if(is_array($res['cpu']['mhz'])) $res['cpu']['mhz'] = implode("<br />", $res['cpu']['mhz']);
            if(is_array($res['cpu']['cache'])) $res['cpu']['cache'] = implode("<br />", $res['cpu']['cache']);
            if(is_array($res['cpu']['bogomips'])) $res['cpu']['bogomips'] = implode("<br />", $res['cpu']['bogomips']);
        }
        //服务器运行时间
        $str = @file("/proc/uptime");
        if(!$str) return false;
        $str = explode(" ", implode("", $str));
        $str = trim($str[0]);
        $min = $str/60;
        $hours = $min/60;
        $days = floor($hours/24);
        $hours = floor($hours-($days*24));
        $min = floor($min-($days*60*24)-($hours*60));
        $res['uptime'] = $days."天".$hours."小时".$min."分钟";
        //内存
        $str = @file("/proc/meminfo");
        if(!$str) return false;
        $str = implode("", $str);
        preg_match_all("/MemTotal\s{0,}\:+\s{0,}([\d\.]+).+?MemFree\s{0,}\:+\s{0,}([\d\.]+).+?Cached\s{0,}\:+\s{0,}([\d\.]+).+?SwapTotal\s{0,}\:+\s{0,}([\d\.]+).+?SwapFree\s{0,}\:+\s{0,}([\d\.]+)/s", $str, $buf);
        preg_match_all("/Buffers\s{0,}\:+\s{0,}([\d\.]+)/s", $str, $buffers);
        $resmem['memTotal'] = round($buf[1][0]/1024, 2);
        $resmem['memFree'] = round($buf[2][0]/1024, 2);
        $resmem['memBuffers'] = round($buffers[1][0]/1024, 2);
        $resmem['memCached'] = round($buf[3][0]/1024, 2);
        $resmem['memUsed'] = $resmem['memTotal']-$resmem['memFree'];
        $resmem['memPercent'] = (floatval($resmem['memTotal'])!=0)?round($resmem['memUsed']/$resmem['memTotal']*100,2):0;
        $resmem['memRealUsed'] = $resmem['memTotal'] - $resmem['memFree'] - $resmem['memCached'] - $resmem['memBuffers']; //真实内存使用
        $resmem['memRealFree'] = $resmem['memTotal'] - $resmem['memRealUsed']; //真实空闲
        $resmem['memRealPercent'] = (floatval($resmem['memTotal'])!=0)?round($resmem['memRealUsed']/$resmem['memTotal']*100,2):0; //真实内存使用率
        $resmem['memCachedPercent'] = (floatval($resmem['memCached'])!=0)?round($resmem['memCached']/$resmem['memTotal']*100,2):0; //Cached内存使用率
        $resmem['swapTotal'] = round($buf[4][0]/1024, 2);
        $resmem['swapFree'] = round($buf[5][0]/1024, 2);
        $resmem['swapUsed'] = round($resmem['swapTotal']-$resmem['swapFree'], 2);
        $resmem['swapPercent'] = (floatval($resmem['swapTotal'])!=0)?round($resmem['swapUsed']/$resmem['swapTotal']*100,2):0;
        $resmem = $this->formatmem($resmem); //格式化内存显示单位
        $res = array_merge($res,$resmem);
        // LOAD AVG 系统负载
        $str = @file("/proc/loadavg");
        if (!$str) return false;
        $str = explode(" ", implode("", $str));
        $str = array_chunk($str, 4);
        $res['loadAvg'] = implode(" ", $str[0]);
        return $res;
    }
    public function sys_freebsd(){ //freeBSD系统探测
        $res['cpu']['num']   = do_command('sysctl','hw.ncpu'); //CPU
        $res['cpu']['model'] = do_command('sysctl','hw.model');
        $res['loadAvg']      = do_command('sysctl','vm.loadavg'); //Load AVG  系统负载
        //uptime
        $buf = do_command('sysctl','kern.boottime');
        $buf = explode(' ', $buf);
        $sys_ticks = time()-intval($buf[3]);
        $min = $sys_ticks/60;
        $hours = $min/60;
        $days = floor($hours/24);
        $hours = floor($hours-($days*24));
        $min = floor($min-($days*60*24)-($hours*60));
        $res['uptime'] = $days.'天'.$hours.'小时'.$min.'分钟';
        //内存
        $buf = do_command('sysctl','hw.physmem');
        $resmem['memTotal'] = round($buf/1024/1024, 2);
        $str = do_command('sysctl','vm.vmtotal');
        preg_match_all("/\nVirtual Memory[\:\s]*\(Total[\:\s]*([\d]+)K[\,\s]*Active[\:\s]*([\d]+)K\)\n/i", $str, $buff, PREG_SET_ORDER);
        preg_match_all("/\nReal Memory[\:\s]*\(Total[\:\s]*([\d]+)K[\,\s]*Active[\:\s]*([\d]+)K\)\n/i", $str, $buf, PREG_SET_ORDER);
        $resmem['memRealUsed'] = round($buf[0][2]/1024, 2);
        $resmem['memCached'] = round($buff[0][2]/1024, 2);
        $resmem['memUsed'] = round($buf[0][1]/1024, 2)+$resmem['memCached'];
        $resmem['memFree'] = $resmem['memTotal']-$resmem['memUsed'];
        $resmem['memPercent'] = (floatval($resmem['memTotal'])!=0)?round($resmem['memUsed']/$resmem['memTotal']*100,2):0;
        $resmem['memRealPercent'] = (floatval($resmem['memTotal'])!=0)?round($resmem['memRealUsed']/$resmem['memTotal']*100,2):0;
        $resmem = $this->formatmem($resmem);
        $res = array_merge($res,$resmem);
        return $res;
    }
    public function do_command($cName, $args){ //执行系统命令FreeBSD
        $cName = empty($cName)?'sysctl':timr($cName);
        if(empty($args)) return false;
        $args = '-n '.$args;
        $buffers = '';
        $command = find_command($cName);
        if(!$command) return false;
        if($fp = @popen("$command $args", 'r')){
            while (!@feof($fp)) {
                $buffers .= @fgets($fp, 4096);
            }
            pclose($fp);
            return trim($buffers);
        }
        return false;
    }
    public function find_command($cName){ //确定shell位置
        $path = array('/bin', '/sbin', '/usr/bin', '/usr/sbin', '/usr/local/bin', '/usr/local/sbin');
        foreach($path as $p) {
            if (@is_executable("$p/$commandName")) return "$p/$commandName";
        }
        return false;
    }
    public function GetCPUUse(){
        $data = @file('/proc/stat');
        $cores = array();
        foreach ($data as $line) {
            if(preg_match('/^cpu[0-9]/', $line)){
                $info = explode(' ', $line);
                $cores[]=array('user'=>$info[1],'nice'=>$info[2],'sys' => $info[3],'idle'=>$info[4],'iowait'=>$info[5],'irq' => $info[6],'softirq' => $info[7]);
            }
        }
        return $cores;
    }
    public function GetCPUPercent($CPU1,$CPU2){
        $num = count($CPU1);
        if($num!==count($CPU2)) return;
        $cups = array();
        for ($i=0; $i < $num; $i++) { 
            $dif = array();
            $dif['user']    = $CPU2[$i]['user'] - $CPU1[$i]['user'];
            $dif['nice']    = $CPU2[$i]['nice'] - $CPU1[$i]['nice'];
            $dif['sys']     = $CPU2[$i]['sys'] - $CPU1[$i]['sys'];
            $dif['idle']    = $CPU2[$i]['idle'] - $CPU1[$i]['idle'];
            $dif['iowait']  = $CPU2[$i]['iowait'] - $CPU1[$i]['iowait'];
            $dif['irq']     = $CPU2[$i]['irq'] - $CPU1[$i]['irq'];
            $dif['softirq'] = $CPU2[$i]['softirq'] - $CPU1[$i]['softirq'];
            $total = array_sum($dif);
            $cpu = array();
            foreach($dif as $x=>$y) 
                $cpu[$x] = round($y/$total*100, 2);
            $cpus['cpu'.$i] = $cpu;
        }
        return $cpus;
    }
    public function GetDisk(){ //获取硬盘情况
        $d['t'] = round(@disk_total_space(".")/(1024*1024*1024),3);
        $d['f'] = round(@disk_free_space(".")/(1024*1024*1024),3);
        $d['u'] = $d['t']-$d['f'];
        $d['PCT'] = (floatval($d['t'])!=0)?round($d['u']/$d['t']*100,2):0;
        return $d;
    }
    private function formatmem($mem){ //格试化内存显示单位
        if(!is_array($mem)) return $mem;
        $tmp = array(
            'memTotal', 'memUsed', 'memFree', 'memPercent',
            'memCached', 'memRealPercent',
            'swapTotal', 'swapUsed', 'swapFree', 'swapPercent'
        );
        foreach ($mem as $k=>$v) {
            if(!strpos($k, 'Percent')){
                $v = $v<1024?$v.' M':$v.' G';
            }
            $mem[$k] = $v;
        }
        foreach ($tmp as $v) {
            $mem[$v] = $mem[$v]?$mem[$v]:0;
        }
        return $mem;
    }
    public function GetNetWork(){ //网卡流量
        $strs = @file("/proc/net/dev");
        $lines = count($strs);
        for ($i=2; $i < $lines; $i++) { 
            preg_match_all( "/([^\s]+):[\s]{0,}(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)/", $strs[$i], $info );
            $res['OutSpeed'][$i] = $info[10][0];
            $res['InputSpeed'][$i] = $info[2][0];
            $res['NetOut'][$i] = $this->formatsize($info[10][0]);
            $res['NetInput'][$i] = $this->formatsize($info[2][0]);
            $res['NetWorkName'][$i] = $info[1][0];
        }
        return $res;
    }
    public function formatsize($size) { //单位转换
        $danwei=array(' B ',' K ',' M ',' G ',' T ');
        $allsize=array();
        $i=0;
        for($i = 0; $i <5; $i++) {
            if(floor($size/pow(1024,$i))==0){break;}
        }
        for($l = $i-1; $l >=0; $l--) {
            $allsize1[$l]=floor($size/pow(1024,$l));
            $allsize[$l]=$allsize1[$l]-$allsize1[$l+1]*1024;
        }
        $len=count($allsize);
        for($j = $len-1; $j >=0; $j--) {
            $fsize=$fsize.$allsize[$j].$danwei[$j];
        }   
        return $fsize;
    }
    public function phpexts(){ //以编译模块
        $able = get_loaded_extensions();
        $str = '';
        foreach ($able as $key => $value) {
            if ($key!=0 && $key%13==0) {
                $str .= '<br />';
            }
            $str .= "$value&nbsp;&nbsp;";
        }
        return $str;
    }
    public function show($varName){ //检测PHP设置参数
        switch($result = get_cfg_var($varName)){
            case 0:
                return '<font color="red">×</font>';
            break;
            case 1:
                return '<font color="green">√</font>';
            break;
            default:
                return $result;
            break;
        }
    }
    public function GetDisFuns(){
        $disFuns=get_cfg_var("disable_functions");
        $str = '';
        if(empty($disFuns)){
            $str = '<font color=red>×</font>';
        }else{ 
            $disFunsarr =  explode(',',$disFuns);
            foreach ($disFunsarr as $key=>$value) {
                if ($key!=0 && $key%8==0) {
                    $str .= '<br />';
                }
                $str .= "$value&nbsp;&nbsp;";
            }
        }
        return $str;
    }
    public function isfun($funName='',$j=0){ // 检测函数支持
        if (!$funName || trim($funName) == '' || preg_match('~[^a-z0-9\_]+~i', $funName, $tmp)) return '错误';
        if(!$j){
            return (function_exists($funName) !== false) ? '<font color="green">√</font>' : '<font color="red">×</font>';
        }else{
            return (function_exists($funName) !== false) ? '√' : '×';
        }
    }
    public function GetGDVer(){
        $strgd = '<font color="red">×</font>';
        if(function_exists(gd_info)) {
            $gd_info = @gd_info();
            $strgd = $gd_info["GD Version"];
        }
        return $strgd;
    }
    public function GetZendInfo(){
        $zendInfo = array();
        $zendInfo['ver'] = zend_version()?zend_version():'<font color=red>×</font>';
        $phpv = substr(PHP_VERSION,2,1);
        $zendInfo['loader'] = $phpv>2?'ZendGuardLoader[启用]':'Zend Optimizer';
        if($phpv>2){
            $zendInfo['html'] = get_cfg_var("zend_loader.enable")?'<font color=green>√</font>':'<font color=red>×</font>';
        }elseif(function_exists('zend_optimizer_version')){
            $zendInfo['html'] = zend_optimizer_version();
        }else{
            $zendInfo['html']= (get_cfg_var("zend_optimizer.optimization_level") ||
                                get_cfg_var("zend_extension_manager.optimizer_ts") ||
                                get_cfg_var("zend.ze1_compatibility_mode") ||
                                get_cfg_var("zend_extension_ts"))?'<font color=green>√</font>':'<font color=red>×</font>';
        }
        return $zendInfo;
    }
    public function GetIconcube(){
        $str = '<font color=red>×</font>';
        if(extension_loaded('ionCube Loader')){
            $ys = ionCube_Loader_version();
            $gm = '.'.(int)substr($ys, 3, 2);
            $str = $ys.$gm;
        }
        return $str;
    }
    public function CHKModule($cName){
        if(empty($cName)) return '错误';
        $str = phpversion($cName);
        return empty($str)?'<font color=red>×</font>':$str;
    }
    public function GetDBVer($dbname){
        if(empty($dbname)) return '错误';
        switch ($dbname) {
            case 'mysql':
                if(function_exists("mysql_get_server_info")){
                    $s = @mysql_get_server_info();
                    $s = $s ? '&nbsp; mysql_server 版本：'.$s:'';
                    $c = @mysql_get_client_info();
                    $c = $c ? '&nbsp; mysql_client 版本：'.$c:'';
                    return $s.$c;
                }
                return '';
                break;
            case 'sqlite':
                if(extension_loaded('sqlite3')){
                    $sqliteVer = SQLite3::version();
                    $str = '<font color=green>√</font>';
                    $str .= 'SQLite3　Ver'.$sqliteVer['versionString'];
                }else{
                    $str = $this->isfun('sqlite_close');
                    if(strpos($str, '√')!==false){
                        $str .= '&nbsp; 版本：'.sqlite_libversion();
                    }
                }
                return $str;
                break;
             
            default:
                return '';
                break;
        }
    }
}
 
$title = 'PHP服务器信息探针';
$j_version = '1.0.0';
$S = new ServerInfo();
$phpSelf = $_SERVER['PHP_SELF'] ? $_SERVER['PHP_SELF'] : $_SERVER['SCRIPT_NAME'];
$disFuns=get_cfg_var("disable_functions");
$disFuns = strpos('phpinfo', needle)?'<font color="red">×</font>':"<a href='$phpSelf?act=phpinfo' target='_blank'>PHPINFO</a>";
$strcookies = isset($_COOKIE)?'<font color="green">√</font>' : '<font color="red">×</font>';
$strsmtp = get_cfg_var("SMTP")?'<font color="green">√</font>' : '<font color="red">×</font>';
$smtpadd = get_cfg_var("SMTP")?get_cfg_var("SMTP"):'<font color="red">×</font>';
 
//ajax调用实时刷新
if ($_GET['act'] == "rt"){
    $arr=array('useSpace'=>$S->hd['u'],
        'freeSpace'=>$S->hd['f'],
        'hdPercent'=>$S->hd['PCT'],
        'barhdPercent'=>$S->hd['PCT'].'%',
        'TotalMemory'=>$S->sysInfo['memTotal'],
        'UsedMemory'=>$S->sysInfo['memUsed'],
        'FreeMemory'=>$S->sysInfo['memFree'],
        'CachedMemory'=>$S->sysInfo['memCached'],
        'Buffers'=>$S->sysInfo['memBuffers'],
        'TotalSwap'=>$S->sysInfo['swapTotal'],
        'swapUsed'=>$S->sysInfo['swapUsed'],
        'swapFree'=>$S->sysInfo['swapFree'],
        'loadAvg'=>$S->sysInfo['loadAvg'],
        'uptime'=>$S->sysInfo['uptime'],
        'freetime'=>"$freetime",
        'bjtime'=>"$bjtime",
        'stime'=>$S->S['sTime'],
        'cpuuse'=>$S->CPU_Use,
        'memRealPercent'=>$S->sysInfo['memRealPercent'],
        'memRealUsed'=>$S->sysInfo['memRealUsed'],
        'memRealFree'=>$S->sysInfo['memRealFree'],
        'memPercent'=>$S->sysInfo['memPercent'].'%',
        'memCachedPercent'=>$S->sysInfo['memCachedPercent'],
        'barmemCachedPercent'=>$S->sysInfo['memCachedPercent'].'%',
        'swapPercent'=>$S->sysInfo['swapPercent'],
        'barmemRealPercent'=>$S->sysInfo['memRealPercent'].'%',
        'barswapPercent'=>$S->sysInfo['swapPercent'].'%',
        'NetOut2'=>$S->NetWork['NetOut'][2],
        'NetOut3'=>$S->NetWork['NetOut'][3],
        'NetOut4'=>$S->NetWork['NetOut'][4],
        'NetOut5'=>$S->NetWork['NetOut'][5],
        'NetOut6'=>$S->NetWork['NetOut'][6],
        'NetOut7'=>$S->NetWork['NetOut'][7],
        'NetOut8'=>$S->NetWork['NetOut'][8],
        'NetOut9'=>$S->NetWork['NetOut'][9],
        'NetOut10'=>$S->NetWork['NetOut'][10],
        'NetInput2'=>$S->NetWork['NetInput'][2],
        'NetInput3'=>$S->NetWork['NetInput'][3],
        'NetInput4'=>$S->NetWork['NetInput'][4],
        'NetInput5'=>$S->NetWork['NetInput'][5],
        'NetInput6'=>$S->NetWork['NetInput'][6],
        'NetInput7'=>$S->NetWork['NetInput'][7],
        'NetInput8'=>$S->NetWork['NetInput'][8],
        'NetInput9'=>$S->NetWork['NetInput'][9],
        'NetInput10'=>$S->NetWork['NetInput'][10],
        'NetOutSpeed2'=>$S->NetWork['OutSpeed'][2],
        'NetOutSpeed3'=>$S->NetWork['OutSpeed'][3],
        'NetOutSpeed4'=>$S->NetWork['OutSpeed'][4],
        'NetOutSpeed5'=>$S->NetWork['OutSpeed'][5],
        'NetInputSpeed2'=>$S->NetWork['InputSpeed'][2],
        'NetInputSpeed3'=>$S->NetWork['InputSpeed'][3],
        'NetInputSpeed4'=>$S->NetWork['InputSpeed'][4],
        'NetInputSpeed5'=>$S->NetWork['InputSpeed'][5]
        );
    $jarr=json_encode($arr); 
    $_GET['callback'] = htmlspecialchars($_GET['callback']);
    echo $_GET['callback'],'(',$jarr,')';
    exit;
}
 
 
function memory_usage() {
    $memory  = ( ! function_exists('memory_get_usage')) ? '0' : round(memory_get_usage()/1024/1024, 2).'MB';
    return $memory;
}
// 计时
function microtime_float() {
    $mtime = microtime();
    $mtime = explode(' ', $mtime);
    return $mtime[1] + $mtime[0];
}
?>
<!DOCTYPE html>
<html>
<head>
<title><?=$title?></title>
<meta http-equiv="X-UA-Compatible" content="IE=EmulateIE7" />
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<style type="text/css">
<!--
* {font-family: "Microsoft Yahei",Tahoma, Arial; }
body{text-align: center; margin: 0 auto; padding: 0; background-color:#fafafa;font-size:12px;font-family:Tahoma, Arial}
h1 {font-size: 26px; padding: 0; margin: 0; color: #333333; font-family: "Lucida Sans Unicode","Lucida Grande",sans-serif;}
h1 small {font-size: 11px; font-family: Tahoma; font-weight: bold; }
a{color: #666; text-decoration:none;}
a.black{color: #000000; text-decoration:none;}
.w_logo{height:25px;text-align:center;color:#333;font-size: 15px; width:13%; }
.j_top{display:table;font-weight:bold;background:#dedede;color:#626262;width: 100%;text-align: left; height: 25px; line-height: 25px;
box-shadow: 1px 1px 1px #CCC;
-moz-box-shadow: 1px 1px 1px #CCC;
-webkit-box-shadow: 1px 1px 1px #CCC;
-ms-filter: "progid:DXImageTransform.Microsoft.Shadow(Strength=2, Direction=135, Color='#CCCCCC')";}
.j_top a{text-align:center; width:8.7%; display: table-cell; padding: 5px 0px;}
.j_top a:hover{background:#dadada;}
.con{width: 90%;margin: 10px auto;}
.con .j_top{ padding: 0px 10px;}
.j_tb{display:table;width:100%;padding:5px 10px;border-bottom:1px solid #CCCCCC;text-align:left;}
.j_td{display:table-cell;}
.j_td_t{width:120px;}
.j_td_c{width:50%;}
.j_td_t1{width:320px;}
 
.w_foot{height:25px;text-align:center; background:#dedede;}
input{padding: 2px; background: #FFFFFF; border-top:1px solid #666666; border-left:1px solid #666666; border-right:1px solid #CCCCCC; border-bottom:1px solid #CCCCCC; font-size:12px}
input.btn{font-weight: bold; height: 20px; line-height: 20px; padding: 0 6px; color:#666666; background: #f2f2f2; border:1px solid #999;font-size:12px}
.bar {border:1px solid #999999; background:#FFFFFF; height:5px; font-size:2px; width:89%; margin:2px 0 5px 0;padding:1px; overflow: hidden;}
.bar_1 {border:1px dotted #999999; background:#FFFFFF; height:5px; font-size:2px; width:89%; margin:2px 0 5px 0;padding:1px; overflow: hidden;}
.barli_red{background:#ff6600; height:5px; margin:0px; padding:0;}
.barli_blue{background:#0099FF; height:5px; margin:0px; padding:0;}
.barli_green{background:#36b52a; height:5px; margin:0px; padding:0;}
.barli_black{background:#333; height:5px; margin:0px; padding:0;}
.barli_1{background:#999999; height:5px; margin:0px; padding:0;}
.barli{background:#36b52a; height:5px; margin:0px; padding:0;}
#page {width: 100%; padding: 0 auto; margin: 0 auto; text-align: left;}
#header{position:relative; padding:5px;}
-->
</style>
<script language="JavaScript" type="text/javascript" src="http://lib.sinaapp.com/js/jquery/1.7/jquery.min.js"></script>
<script type="text/javascript"> 
$(document).ready(function(){getJSONData();});
var OutSpeed2=<?=floor($S->NetWork['OutSpeed'][2]);?>;
var OutSpeed3=<?=floor($S->NetWork['OutSpeed'][3]);?>;
var OutSpeed4=<?=floor($S->NetWork['OutSpeed'][4]);?>;
var OutSpeed5=<?=floor($S->NetWork['OutSpeed'][5]);?>;
var InputSpeed2=<?=floor($S->NetWork['InputSpeed'][2]);?>;
var InputSpeed3=<?=floor($S->NetWork['InputSpeed'][3]);?>;
var InputSpeed4=<?=floor($S->NetWork['InputSpeed'][4]);?>;
var InputSpeed5=<?=floor($S->NetWork['InputSpeed'][5]);?>;
function getJSONData(){
    setTimeout("getJSONData()", 1000);
    $.getJSON('?act=rt&callback=?', displayData);
}
function ForDight(Dight,How){ 
  if (Dight<0){
    var Last=0+"B/s";
  }else if (Dight<1024){
    var Last=Math.round(Dight*Math.pow(10,How))/Math.pow(10,How)+"B/s";
  }else if (Dight<1048576){
    Dight=Dight/1024;
    var Last=Math.round(Dight*Math.pow(10,How))/Math.pow(10,How)+"K/s";
  }else{
    Dight=Dight/1048576;
    var Last=Math.round(Dight*Math.pow(10,How))/Math.pow(10,How)+"M/s";
  }
    return Last; 
}
function displayData(dataJSON){
    $("#useSpace").html(dataJSON.useSpace);
    $("#freeSpace").html(dataJSON.freeSpace);
    $("#hdPercent").html(dataJSON.hdPercent);
    $("#barhdPercent").width(dataJSON.barhdPercent);
    $("#TotalMemory").html(dataJSON.TotalMemory);
    $("#UsedMemory").html(dataJSON.UsedMemory);
    $("#FreeMemory").html(dataJSON.FreeMemory);
    $("#CachedMemory").html(dataJSON.CachedMemory);
    $("#Buffers").html(dataJSON.Buffers);
    $("#TotalSwap").html(dataJSON.TotalSwap);
    $("#swapUsed").html(dataJSON.swapUsed);
    $("#swapFree").html(dataJSON.swapFree);
    $("#swapPercent").html(dataJSON.swapPercent);
    $("#loadAvg").html(dataJSON.loadAvg);
    $("#uptime").html(dataJSON.uptime);
    $("#freetime").html(dataJSON.freetime);
    $("#stime").html(dataJSON.stime);
    $("#bjtime").html(dataJSON.bjtime);
    $("#cpuuse").html(dataJSON.cpuuse);
    $("#memRealUsed").html(dataJSON.memRealUsed);
    $("#memRealFree").html(dataJSON.memRealFree);
    $("#memRealPercent").html(dataJSON.memRealPercent);
    $("#memPercent").html(dataJSON.memPercent);
    $("#barmemPercent").width(dataJSON.memPercent);
    $("#barmemRealPercent").width(dataJSON.barmemRealPercent);
    $("#memCachedPercent").html(dataJSON.memCachedPercent);
    $("#barmemCachedPercent").width(dataJSON.barmemCachedPercent);
    $("#barswapPercent").width(dataJSON.barswapPercent);
    $("#NetOut2").html(dataJSON.NetOut2);
    $("#NetOut3").html(dataJSON.NetOut3);
    $("#NetOut4").html(dataJSON.NetOut4);
    $("#NetOut5").html(dataJSON.NetOut5);
    $("#NetOut6").html(dataJSON.NetOut6);
    $("#NetOut7").html(dataJSON.NetOut7);
    $("#NetOut8").html(dataJSON.NetOut8);
    $("#NetOut9").html(dataJSON.NetOut9);
    $("#NetOut10").html(dataJSON.NetOut10);
    $("#NetInput2").html(dataJSON.NetInput2);
    $("#NetInput3").html(dataJSON.NetInput3);
    $("#NetInput4").html(dataJSON.NetInput4);
    $("#NetInput5").html(dataJSON.NetInput5);
    $("#NetInput6").html(dataJSON.NetInput6);
    $("#NetInput7").html(dataJSON.NetInput7);
    $("#NetInput8").html(dataJSON.NetInput8);
    $("#NetInput9").html(dataJSON.NetInput9);
    $("#NetInput10").html(dataJSON.NetInput10); 
    $("#NetOutSpeed2").html(ForDight((dataJSON.NetOutSpeed2-OutSpeed2),3)); OutSpeed2=dataJSON.NetOutSpeed2;
    $("#NetOutSpeed3").html(ForDight((dataJSON.NetOutSpeed3-OutSpeed3),3)); OutSpeed3=dataJSON.NetOutSpeed3;
    $("#NetOutSpeed4").html(ForDight((dataJSON.NetOutSpeed4-OutSpeed4),3)); OutSpeed4=dataJSON.NetOutSpeed4;
    $("#NetOutSpeed5").html(ForDight((dataJSON.NetOutSpeed5-OutSpeed5),3)); OutSpeed5=dataJSON.NetOutSpeed5;
    $("#NetInputSpeed2").html(ForDight((dataJSON.NetInputSpeed2-InputSpeed2),3));   InputSpeed2=dataJSON.NetInputSpeed2;
    $("#NetInputSpeed3").html(ForDight((dataJSON.NetInputSpeed3-InputSpeed3),3));   InputSpeed3=dataJSON.NetInputSpeed3;
    $("#NetInputSpeed4").html(ForDight((dataJSON.NetInputSpeed4-InputSpeed4),3));   InputSpeed4=dataJSON.NetInputSpeed4;
    $("#NetInputSpeed5").html(ForDight((dataJSON.NetInputSpeed5-InputSpeed5),3));   InputSpeed5=dataJSON.NetInputSpeed5;
}
</script>
</head>
<body>
<a name="j_top"></a>
<div id="page">
    <div class="j_top"><a href="#j_php">PHP参数</a>
        <a href="#j_module">组件支持</a>
        <a href="#j_module_other">第三方组件</a>
        <a href="#j_db">数据库支持</a>
        <a href="#j_performance">性能检测</a>
        <a href="#j_networkspeed">网速检测</a>
        <a href="#j_MySQL">MySQL检测</a>
        <a href="#j_function">函数检测</a>
        <a href="#j_mail">邮件检测</a>
    </div>
    <!--Server info-->
    <div class="con">
        <div class="j_top">服务器参数</div>
        <div class="j_tb"><label class="j_td j_td_t">服务器域名/IP地址</label><label><?=$S->S['DomainIP'];?></label></div>
        <div class="j_tb"><label class="j_td j_td_t">服务器标识</label><label><?=$S->S['Flag'];?></label></div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器操作系统</label><label class="j_td"><?=$S->S['OS'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器解译引擎</label><label class="j_td"><?=$S->S['WebEngine'];?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器语言</label><label class="j_td"><?=$S->S['Language'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器端口</label><label class="j_td"><?=$S->S['WebPort'];?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器主机名</label><label class="j_td"><?=$S->S['Name'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">绝对路径</label><label class="j_td"><?=$S->S['WebPath'];?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">管理员邮箱</label><label class="j_td"><?=$S->S['Email'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">探针路径</label><label class="j_td"><?=$S->S['ProbePath'];?></label></div>
        </div>
    </div>
    <!--Server Real-Time-->
    <div class="con">
        <div class="j_top">服务器实时数据</div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器当前时间</label><label class="j_td" id="stime"><?=$S->S['sTime'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t">服务器已运行时间</label><label class="j_td" id="uptime"><?=$S->sysInfo['uptime'];?></label></div>
        </div>
        <div class="j_tb"><label class="j_td j_td_t">CPU型号[<?=$S->sysInfo['cpu']['num'];?>核]</label><label><?=$S->sysInfo['cpu']['model'];?></label></div>
        <div class="j_tb"><label class="j_td j_td_t">CPU使用情况</label><label id="cpuuse"><?=$S->CPU_Use;?></label></div>
        <div class="j_tb"><label class="j_td j_td_t">硬盘使用状况</label>
            <label>总空间 <?=$S->hd['t'];?>&nbsp;G，已用 <font color='#333333'><span id="useSpace"><?=$S->hd['u'];?></span></font>&nbsp;G，
                空闲 <font color='#333333'><span id="freeSpace"><?=$S->hd['f'];?></span></font>&nbsp;G，
                使用率 <span id="hdPercent"><?=$S->hd['PCT'];?></span>%
                <div class="bar"><div id="barhdPercent" class="barli_black" style="width:<?=$S->hd['PCT']?>%" >&nbsp;</div> </div>
            </label>
        </div>
        <div class="j_tb"><label class="j_td j_td_t">内存使用状况</label>
            <label class="j_td">物理内存：共<font color='#CC0000'><?=$S->sysInfo['memTotal'];?> </font>
                , 已用<font color='#CC0000'><span id="UsedMemory"><?=$S->sysInfo['memUsed']?></span></font>
                , 空闲<font color='#CC0000'><span id="FreeMemory"><?=$S->sysInfo['memFree'];?></span></font>
                , 使用率<span id="memPercent"><?=$S->sysInfo['memPercent'];?></span>
                <div class="bar"><div id="barmemPercent" class="barli_red" style="width:<?=$S->sysInfo['memPercent'];?>%" >&nbsp;</div></div>
            <? if($S->sysInfo['memCached']){?>
                Cache化内存为 <span id="CachedMemory"><?=$S->sysInfo['memCached'];?></span>
                , 使用率<span id="memCachedPercent"><?=$S->sysInfo['memCachedPercent'];?></span>% | 
                Buffers缓冲为  <span id="Buffers"><?=$S->sysInfo['memBuffers'];?></span>
                <div class="bar"><div id="barmemCachedPercent" class="barli_blue" style="width:<?=$S->sysInfo['memCachedPercent'];?>%" >&nbsp;</div></div>
                真实内存使用<span id="memRealUsed"><?=$S->sysInfo['memRealUsed'];?></span>
                , 真实内存空闲<span id="memRealFree"><?=$S->sysInfo['memRealFree'];?></span>
                , 使用率<span id="memRealPercent"><?=$S->sysInfo['memRealPercent'];?></span>%
                <div class="bar_1"><div id="barmemRealPercent" class="barli_1" style="width:<?=$S->sysInfo['memRealPercent'];?>%" >&nbsp;</div></div> 
            <?}?>
            <? if ($S->sysInfo['swapTotal']) {?>
                SWAP区：共<?=$S->sysInfo['swapTotal'];?>
                , 已使用<span id="swapUsed"><?=$S->sysInfo['swapUsed'];?></span>
                , 空闲<span id="swapFree"><?=$S->sysInfo['swapFree'];?></span>
                , 使用率<span id="swapPercent"><?=$S->sysInfo['swapPercent'];?></span>%
                <div class="bar"><div id="barswapPercent" class="barli_red" style="width:<?=$S->sysInfo['swapPercent'];?>%" >&nbsp;</div> </div>
            <?}?>
            </label>
        </div>
        <div class="j_tb"><label class="j_td j_td_t">系统平均负载</label><label class="j_td"><?=$S->sysInfo['loadAvg']?></label></div>
    </div>
    <!--net work-->
    <div class="con">
        <div class="j_top">网络使用状况</div>
        <? 
            $netnum = count($S->NetWork);
            for ($i=2; $i < $netnum; $i++) { ?>
                <div class="j_tb">
                    <label class="j_td" style="width:13%"><?=$S->NetWork['NetWorkName'][$i]?></label>
                    <label class="j_td" style="width:29%">入网：<font color='#CC0000'><span id="NetInput<?=$i;?>"><?=$S->NetWork['NetInput'][$i];?></span></font></label>
                    <label class="j_td" style="width:14%">实时：<font color='#CC0000'><span id="NetInputSpeed<?=$i;?>">0B/s</span></font></label>
                    <label class="j_td" style="width:29%">出网: <font color='#CC0000'><span id="NetOut<?=$i?>"><?=$S->NetWork['NetOut'][$i];?></span></font></label>
                    <label class="j_td" style="width:14%">实时: <font color='#CC0000'><span id="NetOutSpeed<?=$i?>">0B/s</span></font></label>
                </div>
        <?}?>
    </div>
    <!--enbale module-->
    <div class="con">
        <div class="j_top">PHP已编译模块检测</div>
        <div class="j_tb"><label class="j_td j_td_t"><?=$S->phpexts();?></label></div>
    </div>
    <!--enbale module-->
    <a name="j_php"></a>
    <div class="con">
        <div class="j_top">PHP相关参数</div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PHP信息（phpinfo）：</label><label class="j_td"><?=$disFuns;?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PHP版本（php_version）：</label><label class="j_td"><?=PHP_VERSION;?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PHP运行方式：</label><label class="j_td"><?=strtoupper(php_sapi_name());?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">脚本占用最大内存（memory_limit）：</label><label class="j_td"><?=$S::show("memory_limit");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PHP安全模式（safe_mode）：</label><label class="j_td"><?=$S::show("safe_mode");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">POST方法提交最大限制（post_max_size）：</label><label class="j_td"><?=$S::show("post_max_size");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">上传文件最大限制（upload_max_filesize）：</label><label class="j_td"><?=$S::show("upload_max_filesize");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">浮点型数据显示的有效位数（precision）：</label><label class="j_td"><?=$S::show("precision");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">脚本超时时间（max_execution_time）：</label><label class="j_td"><?=$S::show("max_execution_time");?>秒</label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">socket超时时间（default_socket_timeout）：</label><label class="j_td"><?=$S::show("default_socket_timeout");?>秒</label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PHP页面根目录（doc_root）：</label><label class="j_td"><?=$S::show("doc_root");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">用户根目录（user_dir）：</label><label class="j_td"><?=$S::show("user_dir");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">dl()函数（enable_dl）：</label><label class="j_td"><?=$S::show("enable_dl");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">指定包含文件目录（include_path）：</label><label class="j_td"><?=$S::show("include_path");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">显示错误信息（display_errors）：</label><label class="j_td"><?=$S::show("display_errors");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">自定义全局变量（register_globals）：</label><label class="j_td"><?=$S::show("register_globals");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">数据反斜杠转义（magic_quotes_gpc）：</label><label class="j_td"><?=$S::show("magic_quotes_gpc");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">"&lt;?...?&gt;"短标签（short_open_tag）：</label><label class="j_td"><?=$S::show("short_open_tag");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">"&lt;% %&gt;"ASP风格标记（asp_tags）：</label><label class="j_td"><?=$S::show("asp_tags");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">忽略重复错误信息（ignore_repeated_errors）：</label><label class="j_td"><?=$S::show("ignore_repeated_errors");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">忽略重复的错误源（ignore_repeated_source）：</label><label class="j_td"><?=$S::show("ignore_repeated_source");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">报告内存泄漏（report_memleaks）：</label><label class="j_td"><?=$S::show("report_memleaks");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">自动字符串转义（magic_quotes_gpc）：</label><label class="j_td"><?=$S::show("magic_quotes_gpc");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">外部字符串自动转义（magic_quotes_runtime）：</label><label class="j_td"><?=$S::show("magic_quotes_runtime");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">打开远程文件（allow_url_fopen）：</label><label class="j_td"><?=$S::show("allow_url_fopen");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">声明argv和argc变量（register_argc_argv）：</label><label class="j_td"><?=$S::show("register_argc_argv");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Cookie 支持：</label><label class="j_td"><?=$strcookies;?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">拼写检查（ASpell Library）：</label><label class="j_td"><?=$S::isfun("aspell_check_raw");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">高精度数学运算（BCMath）：</label><label class="j_td"><?=$S::isfun("bcadd");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PREL相容语法（PCRE）：</label><label class="j_td"><?=$S::isfun("preg_match");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">PDF文档支持：</label><label class="j_td"><?=$S::isfun("pdf_close");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SNMP网络管理协议：</label><label class="j_td"><?=$S::isfun("snmpget");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">VMailMgr邮件处理：</label><label class="j_td"><?=$S::isfun("vm_adduser");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Curl支持：</label><label class="j_td"><?=$S::isfun("curl_init");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SMTP支持：</label><label class="j_td"><?=$strsmtp;?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SMTP地址：</label><label class="j_td"><?=$smtpadd;?></label></div>
        </div>
        <div class="j_tb">
            <label class="j_td j_td_t1">默认支持函数（enable_functions）：</label><label class="j_td"><a href='<?=$phpSelf;?>?act=Function' target='_blank' class='static'>请点这里查看详细！</a></label>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">被禁用的函数（disable_functions）：</label><label class="j_td"><?=$S::GetDisFuns();?></label></div>
        </div>
    </div>
    <!--组件支持-->
    <a name="j_module"></a>
    <div class="con">
        <div class="j_top">组件支持</div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">FTP支持：</label><label class="j_td"><?=$S::isfun("ftp_login");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">XML解析支持：</label><label class="j_td"><?=$S::isfun("xml_set_object");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Session支持：</label><label class="j_td"><?=$S::isfun("session_start");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Socket支持：</label><label class="j_td"><?=$S::isfun("socket_accept");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Calendar支持</label><label class="j_td"><?=$S::isfun('cal_days_in_month');?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">允许URL打开文件：</label><label class="j_td"><?=$S::show("allow_url_fopen");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">GD库支持：</label><label class="j_td"><?=$S::GetGDVer();?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">压缩文件支持(Zlib)：</label><label class="j_td"><?=$S::isfun("gzclose");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">IMAP电子邮件系统函数库：</label><label class="j_td"><?=$S::isfun("imap_close");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">历法运算函数库：</label><label class="j_td"><?=$S::isfun("JDToGregorian");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">正则表达式函数库：</label><label class="j_td"><?=$S::isfun("preg_match");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">WDDX支持：</label><label class="j_td"><?=$S::isfun("wddx_add_vars");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Iconv编码转换：</label><label class="j_td"><?=$S::isfun("iconv");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">mbstring：</label><label class="j_td"><?=$S::isfun("mb_eregi");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">高精度数学运算：</label><label class="j_td"><?=$S::isfun("bcadd");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">LDAP目录协议：</label><label class="j_td"><?=$S::isfun("ldap_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">MCrypt加密处理：</label><label class="j_td"><?=$S::isfun("mcrypt_cbc");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">哈稀计算：</label><label class="j_td"><?=$S::isfun("mhash_count");?></label></div>
        </div>
    </div>
    <a name="j_module_other"></a>
    <!--第三方组件信息-->
    <div class="con">
        <div class="j_top">第三方组件</div>
        <div class="j_tb"><?$zendInfo = $S::GetZendInfo();?>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Zend版本</label><label class="j_td"><?=$zendInfo['ver'];?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1"><?=$zendInfo['loader']?></label><label class="j_td"><?=$zendInfo['html'];?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">eAccelerator</label><label class="j_td"><?=$S->CHKModule('eAccelerator');?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">ioncube</label><label class="j_td"><?=$S->GetIconcube();?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">XCache</label><label class="j_td"><?=$S->CHKModule('XCache');?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">APC</label><label class="j_td"><?=$S->CHKModule('APC');?></label></div>
        </div>
    </div>
    <a name="w_db"></a>
    <!--db-->
    <div class="con">
        <div class="j_top">数据库支持</div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">MySQL 数据库：</label><label class="j_td"><?=$S::isfun('mysql_close').$S::GetDBVer('mysql');?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">ODBC 数据库：</label><label class="j_td"><?=$S::isfun("odbc_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Oracle 数据库：</label><label class="j_td"><?=$S::isfun("ora_close");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SQL Server 数据库：</label><label class="j_td"><?$S::isfun("mssql_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">dBASE 数据库：</label><label class="j_td"><?=$S::isfun("dbase_close");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">mSQL 数据库：</label><label class="j_td"><?=$S::isfun("msql_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SQLite 数据库：</label><label class="j_td"><?=$S->GetDBVer('sqlite');?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Hyperwave 数据库：</label><label class="j_td"><?=$S::isfun("hw_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Postgre SQL 数据库：</label><label class="j_td"><?=$S::isfun("pg_close"); ?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">Informix 数据库：</label><label class="j_td"><?=$S::isfun("ifx_close");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">DBA 数据库：</label><label class="j_td"><?=$S::isfun("dba_close");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">DBM 数据库：</label><label class="j_td"><?=$S::isfun("dbmclose");?></label></div>
        </div>
        <div class="j_tb">
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">FilePro 数据库：</label><label class="j_td"><?=$S::isfun("filepro_fieldcount");?></label></div>
            <div class="j_td" style="width: 50%"><label class="j_td j_td_t1">SyBase 数据库：</label><label class="j_td"><?=$S::isfun("sybase_close");?></label></div>
        </div>
    </div>
 
    <div class="con">
        <div align="center"><?php $run_time = sprintf('%0.4f', microtime_float() - $time_start);?>Processed in <?php echo $run_time?> seconds. <?php echo memory_usage();?> memory usage.</div>
    </div>
</div>
 
</body>
</html>
```