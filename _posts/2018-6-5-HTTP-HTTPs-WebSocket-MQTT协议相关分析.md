---
layout:     post
title:      HTTP-HTTPs-WebSocket-MQTT协议相关分析
date:       2018-6-5
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 协议
---
>最近项目涉及到一个需求是服务端给手机客户端和web客户端主动推送消息，web客户端中还涉及到微信小程序客户端，小程序客户端要求主动推送使用wss协议，获取数据使用https协议。服务端消息推送采用MQTT协议，MQTT协议实现服务器使用的是Mosquitto，本文对涉及到的几个协议做一个简单的比较分析，并简单记录下nginx的配置，用于备忘。  

_ _ _
### **HTTP与HTTPS、WS与WSS对比分析**  
超文本传输协议HTTP协议被用于在Web浏览器和网站服务器之间传递信息，HTTP协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了Web浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此，HTTP协议不适合传输一些敏感信息，比如：信用卡号、密码等支付信息。为了保证数据传输过程中的安全性，出现了HTTPS协议。  

本文不具体介绍HTTPS协议具体是如何保证安全性的，可以参考：[图解SSL/TLS协议](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)。    

HTTP与HTTPS协议在底层都是基于TCP/IP协议来传输的，只不过HTTPS传输的数据是加密过的，并且考虑到了身份冒充等一些其它可能影响数据传输过程安全性的因素。相比于HTTP协议，HTTPS协议在进行真正的数据传输之前，通话双方需要进行握手协商加密算法等相关事宜，之后基于TCP/IP协议传输的就是加密后的数据了。握手过程可以是：   
<img src="/img/2018-6-5/HTTPSHandShakes.png" width="500" height="500" alt="HTTPS协议握手过程" />    
<center>图1：HTTPS协议握手过程</center>   

握手阶段分成五步。  
* 爱丽丝给出协议版本号、一个客户端生成的随机数（Client random），以及客户端支持的加密方法。
* 鲍勃确认双方使用的加密方法，并给出数字证书、以及一个服务器生成的随机数（Server random）。
* 爱丽丝确认数字证书有效，然后生成一个新的随机数（Premaster secret），并使用数字证书中的公钥，加密这个随机数，发给鲍勃。
* 鲍勃使用自己的私钥，获取爱丽丝发来的随机数（即Premaster secret）。
* 爱丽丝和鲍勃根据约定的加密方法，使用前面的三个随机数，生成"对话密钥"（session key），用来加密接下来的整个对话过程。

WSS也可以理解为在普通的webSocket协议上对传输的数据进行了加密。    

_ _ _
### **HTTP与WebSocket对比分析**
WebSocket是为了解决HTTP协议中存在的通信只能由客户端发起的缺陷而建立的协议。WebSocket与HTTP协议是同级的，都属于应用层协议。  

WebSocket协议握手时借助了HTTP协议(因为此时websocket连接尚未建立，websocket连接建立消息需要借助HTTP协议发送到服务器)，WebSocket消息建立后客户端与服务端之间的数据交互就是借助TCP/IP协议实现的了，只不过传输的数据格式是WebSocket协议中规定的数据格式。  

HTTP消息传输也是借助TCP/IP协议实现的，传输的数据格式是HTTP协议中规定的数据格式。  

关于WebSocket与HTTP之间异同的详细介绍，可参考：[WebSocket 是什么原理？为什么可以实现持久连接？](https://www.zhihu.com/question/20215561l)  

_ _ _
### **MQTT协议java客户端与web客户端实现异同**
MQTT是一个客户端/服务器端架构的发布/订阅模式的消息传输协议。它的设计思想是轻巧、开放、简单、规范，因此易于实现。这些特点使得它对于很多场景来说都是很好的选择，包括受限的环境如机器与机器的通信(M2M)以及物联网环境(IOT),这些场景要求很小的代码封装或者网络带宽非常安规。    

MQTT协议在服务端的实现有很多种，比如activeMQ、Mosquitto等等，在客户端的实现也是如此，比如java版本的实现eclipse/paho.mqtt.java，js版本的实现eclipse/paho.mqtt.javascript。  

MQTT协议实现要求客户端与服务器端建立的是持久连接，在java客户端的实现中是借助于TCP/IP协议实现的，在web客户端的实现中是借助webSocket协议实现的，即java客户端的实现中与服务器交换的数据是经过tcp/ip协议-MQTT协议两层协议包裹之后的数据，而web客户端的实现中与服务器交换的数据是经过tcp/ip协议-webSocket协议-MQTT协议三层协议包裹之后的数据。    

_ _ _
### **配置nginx建立小程序与Mosquitto之间的websocket连接**
小程序要求客户端与服务器之间建立的必须是WSS连接，而项目需求需要建立的是基于websocket协议进行数据传输的MQTT协议，这就产生了一个问题：小程序webSocket发出的数据是不含有请求头Sec-WebSocket-Protocol mqtt的，而这个头部对于MQTT服务端来说是必须得；而且MQTT服务器返回的头部数据中也包含Sec-WebSocket-Protocol选项，而这对于小程序来说是不能接受的。这就要求我们在服务端使用nginx进行请求代理，将客户端的请求数据加上Sec-WebSocket-Protocol mqtt选项再发送给MQTT协议服务器实现(比如Mosquitto);将服务端返回的数据取出Sec-WebSocket-Protocol选项后再发送给小程序客户端。  

nginx配置如下：
```java
http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server { // HTTP代理配置
        listen 80;

        server_name localhost;

        location /static/ {
            alias .../static/;
            autoindex on;
        }

        location / {
                proxy_pass   http://127.0.0.1:8080;
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                # Uncomment to enable naxsi on this location
                # include /etc/nginx/naxsi.rules
        }
    }

    server {
        listen 443; // HTTPS代理配置
        server_name domain;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_certificate /usr/local/nginx/certs/server.pem; // 阿里云域名认证之后给的证书
        ssl_certificate_key /usr/local/nginx/certs/server.key; // 阿里云域名认证之后给的key

        ssl_session_timeout 5m;

        ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4";
        ssl_prefer_server_ciphers on;

        location /static/ {
            alias .../static/;
            autoindex on;
        }
        location /wss/ { // 微信小程序端MQTT协议 webSocket连接配置
             proxy_pass http://yourdomain:port;
             proxy_redirect off;
             proxy_set_header Host yourdomain:port;

             proxy_set_header sec-websocket-protocol mqtt;
             more_clear_headers sec-websocket-protocol;

             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
        }

        location /wssWeb/ { // 普通网页端MQTT协议 webSocket连接配置
             proxy_pass http://yourdomain:port;
             proxy_redirect off;
             proxy_set_header Host yourdomain:port;

             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
        }


        location / {
                proxy_pass   http://127.0.0.1:8080;
        }
      }
}

```
需要注意的是要想让nginx配置文件中more_clear_headers生效，就需要在nginx基础上安装openresty/headers-more-nginx-module模块，通过apt-get install方式安装的nginx是不含有这个模块的，所以需要我们手动编译安装nginx，并在安装的时候加入这个模块。   

关于如何在编译安装nginx时添加openresty/headers-more-nginx-module模块，可以参考：[ubuntu16.04 nginx安装](http://www.cnblogs.com/badboyf/p/6422547.html)，[openresty/headers-more-nginx-module](https://github.com/openresty/headers-more-nginx-module#installation)     

mosquitto配置支持websocket如下：  
```java
// 在mosquitto.conf配置文件中添加如下两行代码即可
listener 9443
protocol websockets
```

还要注意的一点是无论是HTTPS连接还是WSS连接，都是由nginx与客户端建立的，nginx与服务端建立的是HTTP连接和WS连接即可，所以上面mosquitto.conf配置文件中并不需要涉及到WSS的相关配置。  

本部分参考文章：[微信小程序MQTT客户端的坑](https://segmentfault.com/a/1190000012865251)

(完)  

