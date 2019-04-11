###### redis  
```bash
<?php
//如果未修改php.ini下面两行注释去掉
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://127.0.0.1:6379');
session_start();
$_SESSION['sessionid'] = 'this is session content!';
echo $_SESSION['sessionid'];
echo '<br/>';

$redis = new redis();
$redis->connect('127.0.0.1', 6379);
//redis用session_id作为key并且是以string的形式存储
echo $redis->get('PHPREDIS_SESSION:' . session_id());
 ?>
```

###### memcache  
```bash
<?php
ini_set('session.save_handler', 'memcache');
ini_set('session.save_path','tcp://127.0.0.1:11211');
session_start();
if (!isset($_SESSION['session_time'])) {   
 $_SESSION['session_time'] = time();
}
echo "session_time:".$_SESSION['session_time']."<br />";
echo "now_time:".time()."<br />";
echo "session_id:".session_id()."<br />";
?>
```