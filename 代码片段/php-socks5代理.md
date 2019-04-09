```php
Class Pipe extends Threaded
{
    private $client;
    private $remote;
    public function __construct($client, $remote) 
    {
        $this->client = $client;
        $this->remote = $remote; 
    }
    public function run()
    {
        for ( ; ; ) {
                $data = stream_socket_recvfrom($this->client, 4096);
                if ($data === false || strlen($data) === 0) {
                    break;
                } 
                $sendBytes = stream_socket_sendto($this->remote, $data);
                if ($sendBytes <= 0) {
                    break;
                }
        }
        stream_socket_shutdown($this->client, STREAM_SHUT_RD);
        stream_socket_shutdown($this->remote, STREAM_SHUT_WR);
    }
}

Class Client extends Threaded
{
    public $fd;
    public function __construct($fd)
    {
        $this->fd = $fd; 
    }

    public function run()
    {
        $data = stream_socket_recvfrom($this->fd, 2);
        $data = unpack('c*', $data);
        if ($data[1] !== 0x05) {
            stream_socket_shutdown($this->fd, STREAM_SHUT_RDWR);
            echo '协议不正确.', PHP_EOL;
            return;
        }
        $nmethods = $data[2];
        $data = stream_socket_recvfrom($this->fd, $nmethods);
        stream_socket_sendto($this->fd, "\x05\x00");
        $data = stream_socket_recvfrom($this->fd, 4);
        $data = unpack('c*', $data);
        $addressType = $data[4];
        if ($addressType === 0x03) { // domain
            $domainLength = unpack('c', stream_socket_recvfrom($this->fd, 1))[1];
            $data = stream_socket_recvfrom($this->fd, $domainLength + 2);
            $domain = substr($data, 0, $domainLength);
            $port = unpack("n", substr($data, -2))[1];
        } else {
            stream_socket_shutdown($this->fd, STREAM_SHUT_RDWR);
            echo '请使用远程dns解析.', PHP_EOL;
        }

        stream_socket_sendto($this->fd, "\x05\x00\x00\x01\x00\x00\x00\x00\x00\x00");
        echo "{$domain}:{$port}", PHP_EOL;
        $remote = stream_socket_client("tcp://{$domain}:{$port}");
        if ($remote === false) {
            stream_socket_shutdown($this->fd, STREAM_SHUT_RDWR);
            return;
        }

        $pool = $this->worker->pipePool;

        $pipe1 = new Pipe($remote, $this->fd);
        $pipe2 = new Pipe($this->fd, $remote);

        $pool->submit($pipe1);
        $pool->submit($pipe2);
    }
}

class ProxyWorker extends Worker
{
    public $pipePool;
    public function __construct($pipePool)
    {
        $this->pipePool = $pipePool;
    }
}

$server = stream_socket_server('tcp://0.0.0.0:1080', $errno, $errstr);
if ($server === false)
    exit($errstr);

$pipePool = new Pool(200, Worker::class);
$pool = new Pool(50, 'ProxyWorker', [$pipePool]);

for( ; ; ) {
    $fd = @stream_socket_accept($server, 60);
    if ($fd === false)
        continue;
    $pool->submit(new Client($fd));
}
```

测试的话，使用curl即可，对了，目前只支持远程dns解析，为啥呢？因为这个玩具后期可是要实现禾斗学上网的哟：  
curl --socks5-hostname 127.0.0.1:1080 http://ip.cn  
