* 使用`dnssec-keygen -a HMAC-MD5 -b 128 -n USER testkey`命令来生成密钥。  
     dnssec-keygen：用来生成更新密钥。  
     -a HMAC-MD5：采用HMAC-MD5加密算法。  
    -b 128：生成的密钥长度为128位。  
    -n USER testkey：密钥的用户名为testkey。  

* 密钥生成后，会在当前目录下自动生成两个密钥文件***.+157+xxx.key和***.+157+xxx.private。  
* 查看两个密钥文件的内容：  
    ```bash
    cat ***.+157+xxx.key
    cat ***.+157+xxx.private
    ```
* 通过同样的方法生成test2key。
* 添加密钥信息到DNS主配置文件中 
```config
vi /etc/named.conf
key testkey {
algorithm hmac-md5;               //algorithm：指明生成密钥的算法。
secret J+mC6Q29xiOtNEBySR4O1g==;  //secret：指明密钥串。
};
key test2key {
algorithm hmac-md5; 
secret PpRyh6fU1ejnutT+jafXag==; 
};
```
* 将test.com区域中的`allow-update { none; }`中的`none`改成`key testkey`;  
   将`none`改成`key testkey`的意思是指明采用`key testkey`作为密钥的用户可以动态更新`test.com`区域  
   示例如下：  
```config
   key "testkey" {
    algorithm hmac-md5;
    secret "3UHeiKTGzDv+hBpS0A9trg==";
};   
key "test2key" {
    algorithm hmac-md5;
    secret "PpRyh6fU1ejnutT+jafXag==";
};  
 
view "external-test" in {
 match-clients { key testkey; acl1; };  //key testkey; 以及acl1;
 zone "test.com" { 
    type master; 
    file "test.zone"; 
    allow-update { key testkey; };
    //allow-transfer { key testkey; };
 };
};
view "external-test2" in {
 match-clients { key test2key; acl2; };
 zone "test.com" { 
    type master; 
    file "test2.zone"; 
    allow-update { key test2key; };
    //allow-transfer { key testkey; };
 };
};
```
* 总结  
以上示例可以实现通过正确的key和acl对两个view进行nsupdate。但bind的acl匹配原则是："由上而下，匹配即退出"，并且match-clients{}中的参数是"or"的关系。  
当执行nsupdate命令时，执行端（一般都是主dns或管理机）所用的ip地址不应该包含在acl1或者acl2中(最后一个view的match-clients{aclx}可包含也可不包含)，否  
则按照以上原则，执行端主机只能访问包含其ip的view了，其它view的访问都将被拒绝。  
 按照以上配置运行bind，在bind主机上执行nsupdate，命令是:
 
 ```bash
 nsupdate -y test2key:PpRyh6fU1ejnutT+jafXag==
             >zone test.com
             >update add www123.test.com 86400 A 192.168.99.100
             >send
 ```
   返回错误结果：`update failed: REFUSED`  
   错误分析：执行nsupdate的主机地址包含在acl1中， 进入了view "external-test"执行更新，显然key不匹配，被拒绝。  
   解决办法：在view "external-test"的match-clients{}中将执行nsupdate主机的地址（bind主机地址）排除掉，  
   命令：`match-clients{ key testkey; !127.0.0.1; !x.x.x.x; acl1;} //x.x.x.x填写bind主机真实ip地址`