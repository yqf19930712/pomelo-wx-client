# pomelo-wx-client
pomelo微信小程序客户端

具体使用方法和[pomelo-jsclient-websocket](https://github.com/pomelonode/pomelo-jsclient-websocket)一样。

-------
## 2018-3-22 补充一些说明：
1. **小程序基础库在1.7.0以上的，请使用`pomeloclient-over1.7.0.js`，原因是1.7版本之后基础库更改了对websocket的支持方式（详见注释及[微信小程序官方文档](https://mp.weixin.qq.com/debug/wxadoc/dev/api/network-socket.html#wxconnectsocketobject)）；**
2. 微信小程序[只允许发起HTTPS或WSS连接](https://mp.weixin.qq.com/debug/wxadoc/dev/api/api-network.html)（且目标只能是备案了的域名，还不能带端口号），因而需要在服务器上配置TLS，比较简单的work around是在服务器配个Nginx反向代理，下面会举一个例子。

#### （新手向）微信小程序对网络连接的限制及解决方式：
1. 只能访问备案后的域名

  解决方式：买个域名，并备案（备案过程中往往要等待十几天）；  
  
2. 只能通过TLS访问你的域名，即协议必须是HTTPS或WSS

  解决方式：pomelo是支持wss的，但事实上没必要用它。直接在服务器上配个Nginx反向代理即可，详见下面的例子；
  
3. 访问域名时还不能带端口号，只能访问默认的443端口，然而pomelo后台我们是要开多个不同的clientPort的呀

  解决方式：（只是一种曲线救国的方式）客户端访问域名时后面跟不同的目录，服务器还是利用Nginx反向代理来转发到pomelo中配置的不同端口即可，pomelo服务器完全不用改动。

#### 一个例子：
##### 服务端：
假设pomelo中配置了如下服务器：
```
"connector":[
      {
        "id": "connector-server-1",
        "host": "X.X.X.X",
        "port": 4050,
        "clientPort": 3050,
        "frontend": true
      }
    ],
    "core":[
      {
        "id": "core-server-1",
        "host": "X.X.X.X",
        "port": 6050
      }
    ],
    "gate":[
      {"id": "gate-server-1", "host": "X.X.X.X", "clientPort": 3014, "frontend": true}
    ]
```
那么客户端应该在连接gate和connector服务器时分别使用`ws://X.X.X.X:3014`和`ws://X.X.X.X:3050`，但由于微信小程序的限制，我们可以改为请求`wss://域名/gate/`和`wss://域名/conn/`。
比如在游戏客户端的代码中这么写：
```
pomelo.init({
			host: 你的域名,
			port: '/conn/', //注意这里实际上不是端口了，只是我懒得改名了...
			log: true,
		}, function () {
			...
});
```
因为在`pomeloclient-over1.7.0.js`库中，我将发送wss请求的地方写成了这样...
```
var url = 'wss://' + host;//小程序必须用TLS
if(port) {
  url += port;
}//小程序不允许带端口号，只能用默认443。这里的所谓port，只是域名后面跟的子目录
```

然后在服务器上对Nginx做如下配置：（`/etc/nginx/nginx.conf`）  
```
...

server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  你备案后的域名;
        root         /usr/share/nginx/html;

        ssl on;
        ssl_certificate default.d/CA.pem; #你的SSL证书
        ssl_certificate_key default.d/CA.key; #你的SSL证书
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers AESGCM:ALL:!DH:!EXPORT:!RC4:+HIGH:!MEDIUM:!LOW:!aNULL:!eNULL;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

#下面将流量转发到应该去的端口上
	location /gate/ {
		proxy_pass http://localhost:3014;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}
	location /conn/ {
		proxy_pass http://localhost:3050;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}
  
  ...
  
```
配置完重启nginx即可：`nginx -c /etc/nginx/nginx.conf`。

