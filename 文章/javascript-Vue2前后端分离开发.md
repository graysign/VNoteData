###### 安装Yii2
Yii2通过composer-asset-plugin整合bower和npm上的前端库到composer的安装上，如果之前安装过这个插件，就不用安装啦，直接
```bash
composer create-project --prefer-dist yiisoft/yii2-app-basic basic
```
如果没有安装过asset-plugin，就先安装插件，再创建一个新的Yii2程序：
```bash
composer global require"fxp/composer-asset-plugin:^1.2.0"composer create-project --prefer-dist yiisoft/yii2-app-basic basic
```
配置下虚拟主机，以apache为例，打开apache目录下的httpd-vhosts.conf配置文件，添加配置：  
```config
<VirtualHost *:80>    
    ServerAdmin    webmaster@dummy-host.example.com    
    DocumentRoot "D:/devs/web/basic/web"    
    ServerName     basic.backend.local    
    ErrorLog "d:/devs/web/logs/gale.local-error.log"    
    CustomLog "d:/devs/web/logs/gale.local-access.log" common
```
DocumentRoot指向的是刚刚创建的Yii2程序的web目录，ServerName是本地域名，配置好apache配置后，需要再改一下hosts文件，以Win7为例，打开` C:\Windows\System32\drivers\etc\hosts` 在文件最后添加一行
```host
127.0.0.1 basic.backend.local
```

保存，重启Apache，打开浏览器，打开地址` http://basic.backend.local/ `看到熟悉的 Congratulations!大字，说明安装配置完成啦！

###### 配置Yii2支持API调用
* 添加API相关Controller的基础类，这里取名BaseAPIController：
```php
namespace app\controllers;
use yii\filters\ContentNegotiator;
use yii\rest\Controller;
use yii\web\Response;
use Yii;
class BaseAPIController extends Controller{    
    public function behaviors(){        
        $behaviors = parent::behaviors();        
        unset($behaviors['authenticator']);        
        $behaviors['corsFilter'] = [            
            'class' => \yii\filters\Cors::className(),            
            'cors' => [                // restrict access to
                'Access-Control-Request-Method' => ['*'],                // Allow only POST and PUT methods
                'Access-Control-Request-Headers' => ['*'],                // Allow only headers 'X-Wsse'
                'Access-Control-Allow-Credentials' => true,                // Allow OPTIONS caching
                'Access-Control-Max-Age' => 3600,                // Allow the X-Pagination-Current-Page header to be exposed to the browser.
                'Access-Control-Expose-Headers' => ['X-Pagination-Current-Page'],            
            ], ];        
            $behaviors['contentNegotiator'] = [            
                'class' => ContentNegotiator::className(),            
                'formats' => [                
                    'application/json' => Response::FORMAT_JSON            
                 ]];        
         return $behaviors;    
    }
}
```
主要解决两个问题：
     * cors问题，就是跨域调用的问题，这个问题可大可小，展开了还能再写很多，这里不细说了。
     * 返回的数据类型，本例所以请求都返回JSON格式数据。
* 修改接受的请求数据格式
传统的web应用，请求类型通常是x-www-form-urlencoded与multipart/form-data, 对应普通的表单提交，以及文件内容提交，默认Yii2不支持Json格式的请求，需要在config/web.php里修改request组件配置 ：  
```php
'request' => [            
    'cookieValidationKey' => 'Dk7BQ6_hk9YRPMdkaaK6FFwIpa123456',            
    'parsers' => [                
        'application/json' => 'yii\web\JsonParser',                
        'text/json' => 'yii\web\JsonParser',            
    ],        
],
```
* 添加权限验证控制器
```php
namespace app\controllers;
class AuthController extends BaseAPIController{    
    public function actionIndex(){        
        $username = \Yii::$app->request->post('name');        
        $password = \Yii::$app->request->post('password');        
        if($username == "admin" && $password == "admin")        
        {            
            return ['success'=>1,'msg'=>'100-token'];        
        }        
        return ['success'=>0,'msg'=>\Yii::t('erp','Username or password error')];    
    }
}
```
简单起见，不涉及数据库操作，用户名密码如果都符合要求，就返回一个token，这个token的值是100-token,为什么是这个值，请看创建好的Yii2项目的Models目录下的User.php.
* 添加需要授权才能访问的控制器
```php
namespace app\controllers;
use Yii;
use yii\filters\auth\HttpBearerAuth;
class ItemController extends BaseAPIController{    
    public    function behaviors()
    {        
        $behaviors = parent::behaviors();        
        if (Yii::$app->getRequest()->getMethod() !== 'OPTIONS') {            
            $behaviors['authenticator'] = [                
                'class' => HttpBearerAuth::className(),            
            ];        
        }        
        return $behaviors;    
     }
     
     public function actionIndex()
     {        
         return ['success'=>1,'msg'=>'hello'];    
     }
}
```
重点是，这个ItemController只有通过验证，才能访问到Index这个action，这里增加的验证器HttpBearerAuth就是用来验证请求的Header里是否带着刚刚分发的token, 之所以不对OPTIONS请求做验证，是因为OPTIONS请求不带别的信息啊，没法验证。浏览器非要在请求之前发个OPTIONS请求，也没办法。
* 测试
打开浏览器访问`http://basic.backend.local/item` 正常情况下，会返回为授权信息 ：
```html
{"name":"Unauthorized","message":"Your request was made with invalid credentials.","code":0,"status":401,"type":"yii\\web\\UnauthorizedHttpException"}
```





###### 安装Node和vue-cli  
   现在前端没有Node简直寸步难行，所以Node是必须安装的，去官网下个安装包，安装好。然后再安装vue-cli工具。  
###### 创建Vue单页面程序
   运行命令：

```bash
vue init webpack-simple basic_front
```
   会让您输入必要的信息，如项目名称，描述，作者之类的。这里简单起见，一路默认回车就是啦。

```bash
D:\devs\web>vue init webpack-simple basic_front
> Project name basic
> Projectdescription A Vue.js project
> Author elvis_lim 163.com
> Use sass? No   
vue-cli · Generated "basic_front".   To get started:     
cd basic_front     
npm install     
npm run dev
```
###### 运行程序  
   如上面命令行输出的提示，vue-cli创建项目后，还需要npm install安装，然后npm run dev运行，看看能不能成功运行。  
   本例需要用到Vue-router这个官方推荐路由，以及axios这个官方推荐http库。至于官方为什么不推荐原来的vue-resource了，我也不知道，前端喜新厌旧，就是这样的。  

```bash
npm install axios bootstrap iview vue-router --save
```
   细心的朋友会发现多安装了bootstrap 和 iview这两个库，没别的目的，就是想让界面好看点，听说也比较火，安装了吧，反正闲着也是闲着。稍等片刻，安装完毕，`npm run dev`看看效果吧。  
  
###### 开始添加前端逻辑  
   逻辑是这样的，这个前端页面，是登录之后才能用的，未登录用户，直接给跳转到登录页。登录成功后，跳回首页，然后调用`Item/index`接口，获取信息然后显示在页面上。  
   好了，计划完毕，开始写代码。在`src`目录下创建一个`config`目录，加一个`setting.js`, 内容如下：  

```javascript
export default{  
    remoteHost:'http://basic.backend.local',  
    userToken:'tk'
}
```
   基本的配置信息就在这里了，包含了后端服务地址，用户标志的key名称。  
   再在src目录下创建一个`services`目录，加一个`auth.js`,用来辅助判断用户登录状态:  

```javascript
import setting from '../config/setting.js'
export default {
    login(data){
        localStorage.setItem(setting.userToken,data)
    },        // authentication status
   authenticated(){
       var t = localStorage.getItem(setting.userToken);
       return t && t.length > 0;
   },
   getToken(){
       return localStorage.getItem(setting.userToken);
   },
   logout(){
       localStorage.setItem(setting.userToken,"");
   }
}
```
   就四个函数，功能不言而喻。  
   再在services目录下加一个`http.js`，用来包装一下`axios`：  

```javascript
import axios from'axios'
import setting from'../config/setting.js'
import router from'../main.js'
import Auth from'./auth.js'// axios 配置
axios.defaults.timeout = 5000;
axios.defaults.baseURL = setting.remoteHost;// http request 拦截器
axios.interceptors.request.use(    
    config => {        
        if (Auth.authenticated()) {          
            var token = Auth.getToken();          
            config.headers.common["Authorization"] = `Bearer ${token}`;        
        }        
        return config;    
    }, 
    err => {        
        returnPromise.reject(err);    
    }
);
axios.interceptors.response.use(    
    response => {        
        return response;    
    },
    error => {        
        if (error.response) {            
            switch (error.response.status) 
            {               
                    case 401:                    // 401 清除token信息并跳转到登录页面                    
                            Auth.logout()                    
                            router.replace({                        
                                path: 'login',                        
                                query: {redirect: router.currentRoute.fullPath}                    
                            })           
            }        
        }        
        console.log(error);//console : Error: Request failed with status code 402
        return Promise.reject(error)
        }
);
export default axios;
```
   
   重点是在request和response两个拦截器。request里，查看用户是处于登录状态，如果是，那么就在请求的headers里加上 `Bearer ${token};` 这样后端在接受到请求的时候，用`HttpBearerAuth`去校验这个token是不是符合要求，就能够进行用户权限验证啦。当然，不用拦截器也是可以的，但是每个请求都需要加Headers头，会不会很累？  
    response拦截器里，检查返回的状态，如果是401状态码，就是用户没有通过权限验证，那就去登录页面再登录啦。同样的，每个请求也是可以检查状态码的，但是太麻烦，偷懒是美德！
    最后，在src目录下的main.js里把这些工具函数连接起来:  

```javascript
import Vue from'vue'
import App from'./App.vue'
import VueRouter from'vue-router'
import http from'./services/http.js'
import iView from'iview';
import'iview/dist/styles/iview.css';
import'bootstrap/dist/css/bootstrap.css'
import Login from'./components/Login.vue'
import Home from'./components/Home.vue'
Vue.prototype.$http = httpVue.use(VueRouter)
Vue.use(iView)
const router = new VueRouter({ 
    mode: 'history',  
    base: __dirname,  
    routes: [{
        path: '/',      
        component: Home,      
        meta: {        
            requireAuth: true      
        }    
      },{      
        path: '/login',      
        component: Login    
    }]
 })

import Auth from'./services/auth.js';
router.beforeEach((to, from, next) => {    
    if(to.meta.requireAuth && !Auth.authenticated()){      
            next({          
                path: '/login',          
                query: { redirect: to.fullPath }        
            })    
    }else{      
        next()    
    }
})
new Vue({  
    el: '#app',  
    router: router,  
    render: h => h(App)
});
export default router
```
写完这一大串后，发现命令行里报了一堆错，看了看，应该是引入了bootstrap,iview的css，而默认的webpack配置文件没有相应的loader, 于是先安装下这些loader:  
```bash
npm install css-loader style-loader --save-dev
```
然后，修改webpack.config.js文件，给file-loader加上woff|woff2|ttf|eot，因为bootstrap用到了这些字体文件，然后再加上style-loader!css-loader, 修改如下：  
```javascript
{        
    test: /\.(png|jpg|gif|svg|woff|woff2|ttf|eot)$/,        
    loader: 'file-loader',        
    options: {          
        name: '[name].[ext]?[hash]'        
    }      
},{         
    test: /\.css$/,         
    loader: 'style-loader!css-loader'      
}
```
   再次`npm run dev`后，熟悉的首页又回来啦。  
###### 添加路由根节点  
   打开`App.vue`，修改后的`App.vue`如下：  
```javascrip
<template>
    <div id="app">
        <router-view class="view" keep-alive transition transition-mode="out-in"></router-view>
    </div>
</template>
<script>export default{} </script>
```
  其实就是用App这个组件作为总的树根，然后Home和Login两个组件作为路由子组件加载进来。  

###### 添加components目录，再添加`Login.vue`和`Home.vue`  

```javascript
<template>
    《div>
        <div class="row">
            <div class="col-md-12 col-sm-12 col-xs-12 text-center" style="padding-top:10%">
                <p class="login title">Login</p>
            </div>
        </div>
        <Form ref="formInline" :model="formInline" :rules="ruleInline" inline>
            <div class="row">
                <div class="col-md-2 col-sm-2 col-xs-2 text-center col-md-offset-5 col-sm-offset-5 col-xs-offset-5">
                    <Form-item prop="user">
                        <Input type="text" v-model="formInline.user "placeholder="用户名">
                            <i class="ivu-icon ivu-icon-ios-people-outline" slot="prepend"></i>
                        </Input>
                    </Form-item>
                </div>
            </div>
            <div class="row">
                <div class="col-md-2 col-sm-2 col-xs-2 text-center col-md-offset-5 col-sm-offset-5 col-xs-offset-5">
                    <Form-item prop="password">
                        <Input type="password"v-model="formInline.password"placeholder="密码">
                            <Icon type="ios-locked-outline"slot="prepend"></Icon>
                        </Input>
                    </Form-item>
                 </div>
            </div>
            <div class="row">
                <div class="col-md-12 col-sm-12 col-xs-12 text-center">
                    <Form-item >
                        <Button type="primary" @click="handleSubmit('formInline')">登录</Button>
                    </Form-item>
                 </div>
            </div>
       </Form>
    </div>
</template>
<style>
.login-title{  
    font-family: '黑体 Bold', '黑体';  
    font-weight: 700;  
    font-style: normal;  
    font-size: 13px;  
    color: #7B7B7B;
}
</style>
<script>
import Auth from'../services/auth.js'
export default {
    data () {
        return {
            formInline: {
                user: '',
                password: ''
            }, ruleInline: {
                user: [{
                    required: true, 
                    message: '请填写用户名',
                    trigger: 'blur' 
                }],
                password:[{ 
                    required: true, 
                    message: '请填写密码', 
                    trigger: 'blur' 
                }]        
            }    
       }    
   }, 
   methods: {    
       handleSubmit(name) {      
           let obj = {        
               name: this.formInline.user,        
               password: this.formInline.password      
           }      
           if(this.formInline.user.length == 0 || this.formInline.password.length == 0){        
               this.$Message.error("用户名或密码不能为空")        
               return;      
           }      
           this.$http.post('/auth/index', obj).then((res) => {          
               console.log(res);          
               if(res.data.success){            
                   Auth.login(res.data.msg);            
                   this.$router.push({path:'/'})          
                }else{            
                    this.$Message.error(res.data.msg); // 登录失败，显示提示语          
                }        
           }, (err) => {            
               this.$Message.error('请求错误！')        
           })    
   }  
}}
</script>
```

   Home.vue  

```javascript
<template>
    <divclass="content">
        <h1>Home Page</h1>{{msg}}                
        <p v-if="msg.length > 0" @click="logout">logout</p>
    </div>
</template>
<script>
import Auth from'../services/auth.js'
export default {  
    data(){    
        return {      
            msg:""    
        }  
    },  
    created(){    
        this.$http.get('/item').then((res)=>{      
            this.msg = res.data.msg;    
        })  
    },  
    methods:{    
        logout(){      
            Auth.logout()      
            this.$router.push({path:'/login'})    
        }  
    }
}
</script>
```

   添加完毕后，`npm run dev `就可以测试了。按照之前设计的逻辑，首先请求，`http://localhost:8080`, 由于/路由对应的是`Home.vue`,而Home组建创建的时候，调用了一个后端需要权限的API，于是返回401错误，然后被重定向到login去登录。登录的时候，发现报错：  
```bash
XMLHttpRequest cannot loadhttp://basic.backend.local/auth/index. No'Access-Control-Allow-Origin' header ispresenton the requested resource. Origin 'http://localhost:8080'is therefore not allowed access. The response had HTTPstatus code 404
```
   意思是说跨域了，`http://localhost:8080`， 不被后端接受。跨域问题有很多解决方法，本人倾向于用应用服务器配置解决，这里以 Apache为例，在后端的web目录下，加上一.htaccess文件，内容如下:
```htaccess
Header always set Access-Control-Allow-Origin "*"    
Header always set Access-Control-Allow-Headers: "X-Requested-With, Content-Type, Origin, Authorization, Accept, Client-Security-Token, Accept-Encoding"    
Header always set Access-Control-Allow-Methods "POST, GET, OPTIONS"    
RewriteEngine On    
RewriteCond %{REQUEST_METHOD} 
OPTIONS    RewriteRule ^(.*)$ $1 [R=200,L]    
RewriteCond %{REQUEST_FILENAME} !-d    
RewriteCond %{REQUEST_FILENAME} !-f    
RewriteRule . index.php
```

   告诉Apache，不要管跨域问题啦。Nginx的话，添加：  
```config
location / {              
    if ($request_method = OPTIONS ) {                
        add_header Access-Control-Allow-Origin "*";                
        add_header Access-Control-Allow-Methods "*";                
        add_header Access-Control-Allow-Headers "X-Requested-With, Content-Type, Origin, Authorization, Accept, Client-Security-Token, Accept-Encoding";                
        add_header Access-Control-Allow-Credentials "true";                
        add_header Content-Length 0;                
        add_header Content-Type text/plain;                
        return200;              
     }              
     try_files    $uri$uri/ /index.php?$args;          
}
```
   跨越问题其实挺复杂， 建议多看看相关资料，这里就不多说了。加了.htaccess文件后，刷新下页面，发现来到了登录页面，这就对啦。继续用admin，admin登录。相信就能正常啦。  
    