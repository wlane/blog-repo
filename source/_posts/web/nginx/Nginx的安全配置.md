---
title: Nginx的安全配置
tag:
- configuration
categories:
- nginx
---

关于nginx的配置可以参考下列网站：

[nginxconfig](https://www.digitalocean.com/community/tools/nginx)

[ssl-config](https://ssl-config.mozilla.org/)

# 1.常用配置

1. 隐藏应用版本信息

   ~~~nginx
   # 请求响应头里面会包含版本信息，可能攻击者会根据版本号去判断漏洞
   http {
       # 隐藏nginx的版本号
   	server_tokens off;
       # 隐藏upstream服务的版本信息
       proxy_hide_header X-Powered-By;
   }
   ~~~

2. 开启https配置

   ~~~nginx
   server {
   	ssl on;
       ssl_certificate /etc/nginx/ssl/server.crt;
       ssl_certificate_key /etc/nginx/ssl/server.key;
       # ssl会话超时时间
       ssl_session_timeout    1d;
       # 使用cache，以便session重用，提高连接速度，大小可配置，这里大概支持40000
       ssl_session_cache      shared:SSL:10m;
       # 是否客户端需要缓存session ticket，提高性能，但是在多个https站点的情况下会导致错误复用，所以这里一般不开启
       ssl_session_tickets    off;
       # 指定ssl公共密钥交换算法文件，以提高通信的安全性：openssl dhparam -out dhparam 2048
       ssl_dhparam /path/to/dhparam;
       # 支持的ssl协议版本，注意这里支持的版本和nginx以及openssl的版本有关
       ssl_protocols TLSv1.2 TLSv1.3;
       # 支持的加密算法套件，注意这里支持的套件种类和nginx以及openssl的版本有关
       ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
       # 是否有服务器决定采用哪种加密算法，TLSv1.2以上需要关闭，服务端已经不采纳这个参数值
       ssl_prefer_server_ciphers off;
       # 是否需要开启OCSP Stapling
       ssl_stapling on;
       # 在开启OCSP Stapling情况下是否验证证书
       ssl_stapling_verify on;
       # 在开启OCSP Stapling和验证的情况下需要配置
       ssl_trusted_certificate /path/to/root_CA_bundle;
   }
   ~~~

   以上需要注意的是关于OCSP的配置详情可参考：[从无法开启 OCSP Stapling 说起 | JerryQu 的小站 (imququ.com)](https://imququ.com/post/why-can-not-turn-on-ocsp-stapling.html)

   需要关注的是它的启用条件，一是服务器端可以访问CA证书网站，二是CA证书网站的OCSP response值需要包含证书信息。

3. 响应头配置

   ~~~nginx
   server {
       # 启用XSS攻击保护，在检查到XSS攻击时，停止渲染页面
   	add_header X-XSS-Protection          "1; mode=block" always;
       # 禁用资源猜测
   	add_header X-Content-Type-Options    "nosniff" always;
   	# 页面跳转时需不需要带上访问来源
       add_header Referrer-Policy           "no-referrer-when-downgrade" always;
   	# 定义页面可以加载的资源类型，这部分需要调试以便确定页面到底有哪些资源类型
       add_header Content-Security-Policy   "default-src 'self' http: https: ws: wss: data: blob: 'unsafe-inline'; frame-ancestors 'self';" always;
   	# 禁用浏览器的属性和API，这里禁用了浏览器的分析用户浏览数据的功能
       add_header Permissions-Policy        "interest-cohort=()" always;
   	# HSTS,要求只能使用https的方式访问网站，支持子域名，在max-age时间内生效
       add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
   
   }
   ~~~

   关于Referrer-Policy的配置参考：[Referrer Policy 介绍 | JerryQu 的小站 (imququ.com)](https://imququ.com/post/referrer-policy.html)

   关于CSP的配置参考：[Content Security Policy 介绍 | JerryQu 的小站 (imququ.com)](https://imququ.com/post/content-security-policy-reference.html)

   关于更多相关配置可参考：https://sysin.org/blog/security-headers

4. 防止缓冲区溢出

   ~~~nginx
   # 表示客户端请求服务器最大允许大小，可以简单视为上传文件的大小限制
   client_max_body_size 1G；
   # 表示客户端请求body占用缓冲区大小
   client_body_buffer_size 1M;
   # 表示客户端请求头部的缓冲区大小
   client_header_buffer_size 4k;
   # 表示一些比较大的请求头使用的缓冲区数量和大小
   large_client_header_buffers 4 8k;
   
   # 表示读取请求body的超时时间
   client_body_timeout 60;
   # 表示读取客户端请求头的超时时间
   client_header_timeout 10;
   # 参数的第一个值表示客户端与服务器长连接的超时时间，超过这个时间，服务器将关闭连接，可选的第二个参数参数表示Response头中Keep-Alive: timeout=time的time值，这个值可以使一些浏览器知道什么时候关闭连接，以便服务器不用重复关闭，如果不指定这个参数，nginx不会在应Response头中发送Keep-Alive信息
   keepalive_timeout 5 5;
   # 表示发送给客户端应答后的超时时间，Timeout是指没有进入完整established状态，只完成了两次握手，如果超过这个时间客户端没有任何响应，nginx将关闭连接
   send_timeout 10;
   ~~~

# 2.限流配置

1. 默认最大连接数：worker_processes x worker_connections的结果；

2. 使用限流的漏桶算法，使得超过定义的限制会报503错误，涉及的模块如下：

   HttpLimitReqModul：    限制并发请求数
   HttpLimitZoneModule：     限制并发连接数 

   ~~~nginx
   http {
       limit_conn_log_level error;
       limit_conn_status 503;
   	# 使用客户端的ip为key来定义一个叫download_req的zone，大小为10M，用来存储session的数据，限制请求数为每秒20次
       limit_req_zone $binary_remote_addr zone=download_req:10m rate=20r/s;
       # 也是使用客户端的ip为key来定义一个叫download_zone的zone，大小为10M，不过这里不定义其他具体的值
       limit_conn_zone $binary_remote_addr zone=download_zone:10m;
       # 这里使用server_name也就是域名来定义一个域名下总共能接受多少并发连接
       limit_conn_zone $server_name zone=myServerName:10m;
       
       server {
           location /download/ {
               # 应用上面的定义，并且设置burst为5，也就是当突发请求量超过了rate的值，则有一个大小为5的缓冲区可以存放等待请求，但是如果等待请求超过了5个，就会返回503。nodelay，如果不设置该选项，请求依次排队等待，设置nodelay后，将会提供处理（rate+burst）请求的能力，不存在排队情况。
           	limit_req zone=download_req burst=5 nodelay;
            }
        }
   
       server {
           location /download/ {
               # 限制一个ip10个并发连接
               limit_conn download_zone 10;
               # 限制一个连接的带宽值，也就是一个ip最大的速率为10x500k=5M
               limit_rate 500k;
               # 当前域名最多能接受100个并发连接
               limit_conn myServerName 100;
           }
       }
   }
   ~~~
   
   可参考：  [Nginx下limit_req模块burst参数超详细解析_那我懂你意思了的博客-CSDN博客](https://blog.csdn.net/hellow__world/article/details/78658041)
   
   需要明确的一点是：nginx的限流统计基于ms，也就是按照上面20r/s的配置，在每个50ms内只允许一个请求，并不是说每秒内加起来达到20次请求。
   
   这种配置可能导致的一个问题是当访问的过程需要经过多个中间节点，那$binary_remote_addr获取的是最接近nginx的代理ip，而不是正式的客户端ip，这就需要一种获取真实客户端ip的方式：
   
   ~~~shell
   http {
   	# 这里需要知道$http_x_forwarded_for指的是什么
   	map $http_x_forwarded_for $clientRealIp {
   		""  $remote_addr;
   		~^(?P<firstAddr>[0-9\.]+),?.*$	$firstAddr;
   	}
   	# 接下来只需要针对$clientRealIp做相应的限流配置即可
   }
   ~~~
   
   其中有一点要注意的是，一般这里的中间节点指的都是cdn或者代理服务等，不包括防火墙、路由器等等设备。
   
   关于http_x_forwarded_for可以参考：[HTTP 请求头中的 X-Forwarded-For | JerryQu 的小站 (imququ.com)](https://imququ.com/post/x-forwarded-for-header-in-http.html) 

# 3.其他配置

1. 拒绝某些User-Agents，避免特定工具扫描网站

   ~~~nginx
   # LWP::Simple是Perl中模拟浏览器访问的类
   # BBBike可能是一个采集街道数据的网站
   if ($http_user_agent ~* LWP::Simple|BBBike|wget|curl) {
       return 404;
   }
   # 防止爬虫
   if ($http_user_agent ~* "qihoobot|Baiduspider|Googlebot-Modile|Googlebot-Image|Mediapartners-Google|Adsbot-Google|Yahoo! SSlurp China|YoudaoBot|Sosospider|Sogou spider|Sogou web spider|MSNBot")
   {
       return 404;
   }
   ~~~

2. 禁用所有不需要的请求方法

   ~~~nginx
   # 只允许使用get，post和head请求，可以全局生效
   if ($request_method !~ ^(GET|HEAD|POST)$ ) {
   	return 404; 
   }
   # 只在当前location生效
   location / {
   	limit_except GET HEAD POST { 
           deny all; 
       }
   }
   ~~~

3. 白名单设置

   ~~~nginx
   location /admin/ {
       allow 192.168.1.0/24;
       deny all;
   }
   # 当客户端隐藏在多重代理之后
   set $allow false;
   if ($http_x_forwarded_for ~ "108.2.66.[89]") { 
       set $allow true; 
   }
   if ($allow = false) { 
       return 404; 
   }
   ~~~

4. 账号设置

   ~~~nginx
   # 生成一个admin用户的密码文件
   $ htpasswd -c key/auth.key admin
   # 配置
   server {
       location /admin/ {
       auth_basic "please input user&passwd";
       auth_basic_user_file key/auth.key;
       }
   }
   ~~~

 5. 指定dns

    ~~~nginx
    # 指定的dns地址，缓存为60s，也就是用一个域名每隔60s做一下dns查询
    resolver               8.8.8.8 8.8.4.4 valid=60s;
    # dns超时时间
    resolver_timeout       2s;
    ~~~
    
    由于nginx只在启动时对配置的域名做一次dns解析，因此符合以下条件时需要指定dns：
    
    (1). 动态域名，需要实时做dns解析，这个非常符合那种容器之间使用域名的连接，因为他的ip是不固定的；
    
    (2). proxy_pass使用了变量，而变量中使用了域名。

# 4. https在线检测

国内：

[SSL/TLS安全评估报告 (myssl.com)](https://myssl.com/)

国外：

[ssllabs](https://www.ssllabs.com/ssltest/index.html)