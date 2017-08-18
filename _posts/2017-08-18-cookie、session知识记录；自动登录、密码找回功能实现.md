---
layout:     post
title:      cookie、session知识记录；自动登录、密码找回功能实现
date:       2017-08-18
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Web杂记
---
本文先记录了cookie、session使用的知识点，区分了一些易混淆的地方，之后应用cookie、session实现了自动登录、密码找回功能。  

_ _ _
### **1. cookie、session知识记录**
&emsp;&emsp;(1)cookie存于客户端，session存于服务器端，cookie超时时间默认为浏览器关闭即失效，tomcat中session默认为20分钟失效。  
&emsp;&emsp;(2)cookie超时时间计算方法为自cookie生成时间起到当前时间，session超时时间计算方法为此session自最后一次使用时间起到当前时间。比如cookie生成时间为某天8点整，设置存活时间为4个小时，则在8点到12点之间cookie有效，超过12点，cookie失效；session生成时间也为某天8点整，存活时间30分钟，若用户在此30分钟内未使用此session，则在8点30失效，若用户在8点10分使用了session进行了某些操作，则session失效时间变为了8点40。  
&emsp;&emsp;(3)浏览器多窗口可能共享同一个session，所以session中存储对象的话，要注意线程安全问题。session.set(get)Attribute()方法本身是线程安全的，但是从session中取出来的对象使用时要注意线程安全问题。比如从session中取出一个user对象，之后对此user对象进行操作，要注意user对象中的线程间共享数据(比如hashmap)的线程安全性问题。  
&emsp;&emsp;(4)服务器通过客户端cookie中存储的jsessionId来判断当前用户对应的session(以tomcat为例)，若客户端禁用cookie，必须通过在前端向后端请求的url中添加 ;JSESSIONID=××× 方式来获取session。  
&emsp;&emsp;(5)cookie.setPath()方法可限制cookie的访问路径；cookie.setPath("/")限制当前域名下所有web应用的url路径均可访问此cookie,cookie.setPath("/webapp1/")限制当前域名下仅webapp1应用下的url路径可访问此cookie。  
&emsp;&emsp;(6)cookie默认不可跨域使用，不同域名之间，同一域名的两子域名之间不共享cookie。cookie.setdomain(".abc.com")可使得所有以.abc.com结尾的域名共享cookie;要想使不同域名间共享cookie，则需在服务器端为两个域名分别生成cookie，保证两个cookie内容完全相同。  
&emsp;&emsp;(7)cookie中存储的JSESSIONID一项默认在浏览器关闭时即失效，若想改变其失效时间，只需向cookie中添加同名key-value，并设置访问路径为当前web应用即可。  

_ _ _
### **2. 自动登录功能实现**
&emsp;&emsp;理论上来讲cookie与session均可实现自动登录功能，自动登录要求时间一般较长，比如7天，一个月等等。session存储于服务端，会占用服务器端存储空间，不适合长时间存储，cookie存储于客户端无此限制，所以自动登录功能常采用cookie来实现。  
&emsp;&emsp;我列举了自动登录功能需满足的常见要求如下：  
* 根据cookie中存储的内容可唯一确定一个用户  
* 根据cookie中存储的内容可确定过期时间  
* cookie中存储的内容要尽可能少的暴露用户的信息  
* 用户更改相关信息后可使得此用户之前所有cookie失效  
* 支持同一账号单用户登录或同一账号多用户登录  

&emsp;&emsp;为满足以上需求，设计存储自动登录cookie相关信息表persist_login如下：  
```sql
    'CREATE TABLE `persist_login` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `uuid` varchar(50) NOT NULL,
      `expiretime` datetime NOT NULL,
      `uid` int(11) NOT NULL,
      `sessionid` varchar(50) DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `fk_persist_login_1_idx` (`uid`),
      CONSTRAINT `fk_persist_login_1` FOREIGN KEY (`uid`) REFERENCES `user` (`id`) ON   DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8'
```
&emsp;&emsp;uuid保存一个服务器随机生成的尽量唯一的字符串；uid保存此条记录对应的user的id；expiretime保存过期时间；sessionid保存对应用户最新一次登录系统产生的session对应的Id(此属性为了支持单用户登录)。  
&emsp;&emsp;客户端cookie中保存的信息为：MD5或SHA256（uuid + uid + salt）,salt为服务器端存储在配置文件中的一串随机数。使用uuid与salt的好处有：  
* 增加散列信息复杂度，使得散列后的信息更难被破解  
* 通过更换服务器端配置文件中保存的salt可使得之前产生的cookie一次性全部失效  
* 通过uuid可实现同一账号同时只允许单用户登录  
* 即使cookie中信息被破解，也仅暴露了用户的id信息  
* 更改用户相关信息后可以通过更新此用户对应的uuid来使得之前的cookie失效  

&emsp;&emsp;实现自动登录的流程为：  
&emsp;&emsp;(1)拦截器中获取用户session,判断session中是否有代表此用户已登录的标识，有则放行，没有则判断用户的cookie中信息，若没有cookie或cookie中存储的内容不满足条件，则跳转到登录页面；若cookie中存储的信息满足自动登录条件，则证明此用户之前的session失效，在session中重新设置此用户已登录的标识。  
&emsp;&emsp;(2)若支持多用户登录，则登录页面中用户输入用户名密码重新登录后在persist_login表中添加一项记录即可；若仅支持单用户登录，则登录页面中用户输入用户名密码重新登录后在persist_login表要对此用户之前的记录进行更新，主要更新的是uuid，sessionId，expiretime；检查之前sessionId对应的session（此sessionId可能被复用），若此session有效且session中存储的确实是当前用户已经登录的信息，则将此session设置为无效。  

&emsp;&emsp;此种限制单用户登录的方法要求两用户必须都通过用户名密码登录的方式才有效。比如A用户登陆后，B用户盗取A用户的cookie之后是可以登录的，这就会出现多用户同时登录的情况。防止cookie被盗取可以设置cookie的httponly属性。  

&emsp;&emsp;自动登录流程中提到了需要根据sessionId获取session的需求，这需要下面的一个工具类：  
```java
    import javax.servlet.http.HttpSession;
    import java.util.concurrent.ConcurrentHashMap;

    /**
     * Created by michael on 17-8-18.
     *
     */
    public class MySessionContext {
        private ConcurrentHashMap<String, HttpSession> map = new ConcurrentHashMap<>();

        private static class MySessionContextHolder {//单例模式
            private static final MySessionContext INSTANCE = new MySessionContext();
        }

        private MySessionContext() {}

        public static MySessionContext getInstance() {
            return MySessionContextHolder.INSTANCE;//外部类访问了内部类的private变量
        }

        public void addSession(HttpSession session) {
            if (session == null)
                return;
            map.put(session.getId(), session);
        }

        public HttpSession getSession(String sessionId) {
            if (sessionId == null)
                return null;
            return map.get(sessionId);
        }

        public void delSession(String sessionId) {
            if (sessionId == null)
                return;
            map.remove(sessionId);
        }
    }
    

    import javax.servlet.http.HttpSessionEvent;
    import javax.servlet.http.HttpSessionListener;

    /**
     * Created by michael on 17-8-18.
     *
     */
    public class SessionListener implements HttpSessionListener {
        private MySessionContext sessionContext = MySessionContext.getInstance();

        @Override
        public void sessionCreated(HttpSessionEvent httpSessionEvent) {
            sessionContext.addSession(httpSessionEvent.getSession());
        }

        @Override
        public void sessionDestroyed(HttpSessionEvent httpSessionEvent) {
            sessionContext.delSession(httpSessionEvent.getSession().getId());
        }
    }  


    public static void main(String[] args) {
        String sessionId = "对应的sessionID";
        HttpSession session = MySessionContext.getInstance().getSession(sessionId);
    }

```

_ _ _
### **3. 找回密码功能实现**
&emsp;&emsp;以向注册邮箱发送邮件为例，找回密码有以下需求：
* 每个用户对应的有效的更改密码的邮箱链接只应该有一个
* 链接中存储的验证信息尽可能少的暴露用户信息
* 根据链接中存储的验证信息可知道此链接的有效时间
* 用户点击链接后链接失效

&emsp;&emsp;类似于自动登录，邮箱验证链接中存储的验证信息可以为uuid + uid + salt,数据库中验证找回密码相关的表业余自动登录类似。  
&emsp;&emsp;需要注意的是验证邮箱验证信息是否有效的数据库操作与更改密码的数据库操作应放在同一个事务操作中，防止在更改密码期间用户重新发送了激活邮件更改密码导致的事务不一致性问题。

_ _ _
### **4、登录注册功能的安全性**
&emsp;&emsp;登录注册功能需注意的核心安全问题即用户密码的保护。  
&emsp;&emsp;在用户登录注册时用户输入的密码最好不要直接明文传输，比如可以通过RSA加密方式传输。客户端以公钥对密码进行加密，服务器解密后对密码进行相应处理后存入数据库。若有人窃听到了公钥加密后的密码，虽然可以登录进系统，但毕竟用户的真实密码没有泄露，防止了使用用户真实密码登录进其他用户使用的系统中。  
&emsp;&emsp;当然，上面说法是当使用http协议传输用户密码，若使用https协议传输信息，则无需再进行加密了。  
&emsp;&emsp;在服务端存储用户密码时可采用 密码随机加盐 后散列的方式保存，数据库中存储用户的散列后的密码和使用的盐，随机加盐可防止数据库中信息泄露后攻击者采用查表法大量破解用户密码。  
&emsp;&emsp;关于密码随机加盐为何可防止数据库泄漏时密码被大量破解，可参考以下两篇文章：  
&emsp;&emsp;&emsp;&emsp;[为什么要在密码里加点“盐”（浅显易懂）](https://libuchao.com/2013/07/05/password-salt)  
&emsp;&emsp;&emsp;&emsp;[如何安全的存储用户的密码 (较长较专业)](http://www.freebuf.com/articles/web/28527.html)  
