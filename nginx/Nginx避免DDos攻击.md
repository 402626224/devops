      分布式拒绝服务攻击（DDoS）指的是通过多台机器向一个服务或者网站发送大量看似合法的数据包使其网络阻塞、资源耗尽从而不能为正常用户提供正常服务的攻击手段。随着互联网带宽的增加和相关工具的不断发布，这种攻击的实施难度越来越低，有大量IDC托管机房、商业站点、游戏服务商一直饱受DDoS攻击的困扰，那么如何缓解甚至解决DDoS呢？最近Rick Nelson在Nginx的官方博客上发表了一篇文章，介绍了如何通过Nginx和Nginx Plus缓和DDoS攻击。

      Rick Nelson首先介绍了DDoS攻击的一些特点,例如攻击的流量通常来源于一些固定的IP地址，每一个IP地址会创建比真实用户多得多的连接和请求；同时由于流量全部是由机器产生的，其速率要比人类用户高的多。此外，进行攻击的机器其User-Agent头也不是标准的值，Referer头有时也会被设置成能够与攻击关联起来的值。针对这些特点，Rick Nelson认为Nginx和Nginx Plus有很多能够通过调节或控制流量来应对或者减轻DDoS攻击的特性。

      防御DDOS是一个系统工程，攻击花样多，防御的成本高瓶颈多，防御起来即被动又无奈。DDOS的特点是分布式，针对带宽和服务攻击，也就是四层流量攻击和七层应用攻击，相应的防御瓶颈四层在带宽，七层的多在架构的吞吐量。对于七层的应用攻击，我们还是可以做一些配置来防御的，例如前端是Nginx，主要使用nginx的http_limit_conn和http_limit_req模块来防御。ngx_http_limit_conn_module 可以限制单个IP的连接数，ngx_http_limit_req_module 可以限制单个IP每秒请求数，通过限制连接数和请求数能相对有效的防御CC攻击。下面是配置方法：

 

1、限制每秒请求数
      将Nginx和Nginx Plus可接受的入站请求率限制为某个适合真实用户的值。

      ngx_http_limit_req_module模块通过漏桶原理来限制单位时间内的请求数，一旦单位时间内请求数超过限制，就会返回503错误。配置需要在两个地方设置：

nginx.conf的http段内定义触发条件，可以有多个条件
在location内定义达到触发条件时nginx所要执行的动作
例如：

http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s; //触发条件，所有访问ip 限制每秒10个请求
    ...
    server {
        ...
        location  ~ \.php$ {
            limit_req zone=one burst=5 nodelay;   //执行的动作,通过zone名字对应
               }
           }
     }
参数说明：

$binary_remote_addr  二进制远程地址
zone=one:10m    定义zone名字叫one，并为这个zone分配10M内存，用来存储会话（二进制远程地址），1m内存可以保存16000会话
rate=10r/s;     限制频率为每秒10个请求
burst=5         允许超过频率限制的请求数不多于5个，假设1、2、3、4秒请求为每秒9个，那么第5秒内请求15个是允许的，
反之，如果第一秒内请求15个，会将5个请求放到第二秒，第二秒内超过10的请求直接503，类似多秒内平均速率限制。
nodelay         超过的请求不被延迟处理，设置后15个请求在1秒内处理。
 

限制连接的数量 
      将某个客户端IP地址所能打开的连接数限制为真实用户的合理值。例如，限制每一个IP对网站/store部分打开的连接数不超过10个：

limit_conn_zone $binary_remote_addr zone=addr:10m;
server {
    ...
    location /store/ {
        limit_conn addr 10;
        ...
    }
}
      该配置中，limit_conn_zone指令配置了一个名为addr的共享内存zone用来存储 $binary_remote_addr的请求，location块中/store/的limit_conn指令引用了共享内存zone，并将最大连接数设置为10.

      ngx_http_limit_conn_module的配置方法和参数与http_limit_req模块很像，参数少，要简单很多

http {
    limit_conn_zone $binary_remote_addr zone=addr:10m; //触发条件
    ...
    server {
        ...
        location /download/ {
            limit_conn addr 1;    // 限制同一时间内1个连接，超出的连接返回503
                }
           }
     }
 

关闭慢连接 
      关闭那些一直保持打开同时写数据又特别频繁的连接，因为它们会降低服务器接受新连接的能力。Slowloris就是这种类型的攻击。对此，可以通过client_body_timeout和client_header_timeout指令控制请求体或者请求头的超时时间，例如，通过下面的配置将等待时间控制在5s之内：

server {

    client_body_timeout 5s;

    client_header_timeout 5s;

}

设置IP黑名单 
如果能识别攻击者所使用的客户端IP地址，那么通过deny指令将其屏蔽，让Nginx和Nginx Plus拒绝来自这些地址的连接或请求。例如，通过下面的指令拒绝来自123.123.123.3、123.123.123.5和123.123.123.7的请求：

location / {

    deny 123.123.123.3;

    deny 123.123.123.5;

    deny 123.123.123.7;

}

设置IP白名单 
如果允许访问的IP地址比较固定，那么通过allow和deny指令让网站或者应用程序只接受来自于某个IP地址或者某个IP地址段的请求。例如，通过下面的指令将访问限制为本地网络的一个IP段：

location / {

    allow 192.168.1.0/24;

    deny all;

}

通过缓存削减流量峰值 
      通过启用缓存并设置某些缓存参数让Nginx和Nginx Plus吸收攻击所产生的大部分流量峰值。例如，通过proxy_cache_use_stale指令的updating参数告诉Nginx何时需要更新过期的缓存对象，避免因重复发送更新请求对后端服务器产生压力。

      另外，proxy_cache_key指令定义的键通常会包含嵌入的变量，例如默认的键$scheme$proxy_host$request_uri包含了三个变量，如果它包含$query_string变量，那么攻击者可以通过发送随机的query_string值来耗尽缓存，因此，如果没有特别原因，不要在该键中使用$query_string变量。

阻塞请求 
配置Nginx和Nginx Plus阻塞以下类型的请求：

以某个特定URL为目标的请求
User-Agent头中的值不在正常客户端范围之内的请求
Referer头中的值能够与攻击关联起来的请求
其他头中存在能够与攻击关联在一起的值的请求
例如，通过下面的配置阻塞以/foo.php为目标的攻击：

location /foo.php {

    deny all;

}

或者通过下面的配置阻塞已识别出的User-Agent头的值是foo或者bar的DDoS攻击：

location / {

    if ($http_user_agent ~* foo|bar) {

        return 403;

    }

}

限制对后端服务器的连接数 
      通常Nginx和Nginx Plus实例能够处理比后端服务器多得多的连接数，因此可以通过Nginx Plus限制到每一个后端服务器的连接数。例如，通过下面的配置限制Nginx Plus和每一台后端服务器之间建立的连接数不多于200个：

upstream website {

    server 192.168.100.1:80 max_conns=200;

    server 192.168.100.2:80 max_conns=200;

    queue 10 timeout=30s;

}

 

Byte Range 避免Dos攻击
      Byte Range允许客户端向服务器请求一部分文件，而不是整个文件。大部分支持多线程下载和断点下载的软件都是用的这个功能。这个时候服务器返回的Http Code是206 Partial Requests.

      但是Nginx做反代的时候，如果没有好好的设置，这个功能可能会引来Dos攻击。

      因为默认做反代的时候，Nginx向后端服务器请求的时候是不会把 Range 参数加上的，而是会去请求整个文件，比方说有一个1G的文件，每次请求1M，Nginx会在每次请求的时候去后端请求一个完整的1G文件，然后取出其中的1M发给客户端，这个时候中间的流量会暴增，导致整个服务器宕机。今天因为这个问题导致我检查了很久。

解决方案也很简单，把 Range 加到Header里就行了。

proxy_set_header Range $http_range;
proxy_set_header If-Range $http_if_range;
proxy_no_cache $http_range $http_if_range;
      另外，Rick Nelson还提到了如何处理基于范围的攻击和如何处理高负载的问题，以及如何使用Nginx Plus Status模块发现异常的流量模式，定位DDoS攻击。

 

其它一些防CC的方法
1.Nginx模块 ModSecurity、http_guard、ngx_lua_waf

ModSecurity 应用层WAF，功能强大，能防御的攻击多，配置复杂
ngx_lua_waf 基于ngx_lua的web应用防火墙，使用简单，高性能和轻量级
http_guard 基于openresty
2.软件+Iptables

fail2ban 通过分析日志来判断是否使用iptables拦截
DDoS Deflate 通过netstat判断ip连接数，并使用iptables屏蔽
开头说过抗DDOS是一个系统工程，通过优化系统和软件配置，只能防御小规模的CC攻击，对于大规模攻击、四层流量攻击、混合攻击来说，基本上系统和应用软件没挂，带宽就打满了。下面是我在工作中使用过的防御DDOS的方式：

高防服务器和带流量清洗的ISP 通常是美韩的服务器，部分ISP骨干供应商有流量清洗服务，例如香港的PCCW。通常可以防御10G左右的小型攻击
流量清洗服务 例如：akamai(prolexic),nexusguard 我们最大受到过80G流量的攻击，成功被清洗，但是费用非常贵
CDN 例如：蓝讯 网宿 cloudflare 等，CDN针对DDOS的分布式特点，将流量引流分散，同时对网站又有加速作用，效果好，成本相对低。
总结一下：发动攻击易，防御难。七层好防，四层难防；小型能防，大型烧钱

具体系统设置和nginx更好的防御CC方法，见nginx防御ddos第二篇： http://www.52os.net/articles/nginx-anti-ddos-setting-2.html

 

限制Http body大小避免DDoS攻击
通过在Web服务器中强制执行Http Body限制，避免大型HTTP请求主体导致的DDoS攻击

限制Web服务器中的请求正文大小
通过限制Web服务器配置中的最大请求主体大小，可以非常轻松地缓解此类问题。 事实上，做这样的事情几乎总是一个好主意，因为：
默认情况下，典型的Django / Ruby / PHP框架通常不会优化处理大量数据。 允许用户默认将大量数据传递给它几乎总是一个坏主意。
如果您允许用户将文件上传到您的服务器（如图像或头像），这也可以帮助您防止大文件上传DDoS攻击。
还有许多其他应用程序级别的DDoS攻击可以通过限制输入大小来减轻或防止。 不，这不是写一些不做任何检查的草率代码的借口。 这只是一个额外的保护和安全层。
请记住，安全性最总要。

现在Apache Body大小
<VirtualHost *:8000>
    # other stuff
 
    # Small, safe default (1 MB)
    <Location />
        LimitRequestBody 1048576
    </Location>
 
    # 2 MB
    <Location /users/profile/edit/avatar>
        LimitRequestBody 2097152
    </Location>
 
    # 5 MB
    <Location /users/profile/edit/images>
        LimitRequestBody 5242880
    </Location>
</VirtualHost>
限制Nginx body大小
server {
    # other stuff
    client_max_body_size 1m;
 
    location /users/profile/edit/avatar {
        client_max_body_size 2m;
        # other stuff such as proxy_pass, etc.
    }
 
    location /users/profile/edit/images {
        client_max_body_size 5m;
        # other stuff such as proxy_pass, etc.
    }
}
Limit HTTP Request size
     Similarly, large buffer values or large HTTP requests size make DDoS attacks easier. So, we limit the following buffer values in the Nginx configuration file to mitigate DDoS attacks.

    client_body_buffer_size 200K;
    client_header_buffer_size 2k;
    client_max_body_size 200k;
    large_client_header_buffers 3 1k;

 

Why do we have to care?
      One would be just fine putting the large client_max_body_size in server {} context and be happy that things work.     

      However, this exposes some risk to the DDoS. Many large file uploads will consume disk space on your server. They will consume network bandwidth as well as TCP sockets.

      So what is the rule of thumb here? It depends on your use case.

      First, if you are certain that your website will not accept large file uploads exceeding the default 1MB, do not specify any extra configuration. Let the default value apply.

      If you only accept large uploads directly at PHP URLs which don’t involve any internal rewriting by nginx, you can put the client_max_body_size in the ~ \.php$ location only and this would be most safe.

      Finally, if you accept uploads exceeding 1MB at SEO friendly URLs (e.g. /user/upload), then you can put client_max_body_size 64M; in server {} context. But to address security concerns you might want to take a different approach, which we are going to detail next.

Increase file upload limit in nginx for SEO-friendly upload endpoints.
      So your upload must be taken at a SEO friendly location, e.g. /user/upload and you want to increase the upload limit just there. This would be quite better in terms of security, since other locations will be less prone to large uploads DDoS.

      You can achieve this by specifying a separate location for your “pretty upload handler”, like the following:

server {  
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
    location = /user/upload {
        client_max_body_size 64M;
        fastcgi_pass ...
        fastcgi_param SCRIPT_FILENAME $document_root/index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_NAME /index.php;
    }
    location ~ \.php$ {
         fastcgi_pass ...
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;         
    }
}

      What we did there is relaxed the upload limit just for our upload handler. Note that we have to adjust fastcgi_param SCRIPT_FILENAME and point it directly to /index.php script as there is no longer any rewrite to it. We have essentially converted an indirect match (our initial configuration) into direct match to our upload handler with the pretty URL.

      Note how we put fastcgi_param SCRIPT_NAME /index.php at the end of /user/upload location. This is so it overrides the one which is set in fastcgi_params file.


链接：

https://www.jianshu.com/p/163c26766a80
https://www.52os.net/articles/nginx-anti-ddos-setting.html

http://www.tianwaihome.com/2015/03/nginx-proxy-cache.html

https://bobcares.com/blog/nginx-ddos-prevention/

https://www.tomaz.me/2013/09/15/avoiding-ddos-attacks-caused-by-large-http-request-bodies-by-enforcing-a-hard-limit-in-your-web-server.html

https://www.getpagespeed.com/server-setup/nginx/nginx-client_max_body_size-inheritance

https://www.cyberciti.biz/tips/linux-unix-bsd-nginx-webserver-security.html

https://security.stackexchange.com/questions/94510/any-security-risk-with-raising-client-max-body-size-nginx

 
————————————————
版权声明：本文为CSDN博主「zzhongcy」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zzhongcy/article/details/88735174