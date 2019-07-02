---
title: Nginx 部署
date: "2018-05-06 22:00:00"
categories:
- 开发工具
- nginx
tags:
- true
toc: true
typora-root-url: ..\..\..\
---

### Nginx 命令集

**nginx 命令集提示**

```shell
nginx [-?hvVtq] [-s signal] [-c filename] [-p prefix] [-g directives]

-?,-h           : 打开帮助信息
-v              : 显示版本信息并退出
-V              : 显示版本和配置选项信息，然后退出
-t              : 检测配置文件是否有语法错误，然后退出
-q              : 在检测配置文件期间屏蔽非错误信息
-s signal       : 给一个 nginx 主进程发送信号：stop（停止）, quit（退出）, reopen（重启）, reload（重新加载配置文件）
-p prefix       : 设置前缀路径（默认是：/usr/local/Cellar/nginx/1.2.6/）
-c filename     : 设置配置文件（默认是：/usr/local/etc/nginx/nginx.conf）
-g directives   : 设置配置文件外的全局指令
```

**nginx 启动**

```shell
nginx -c /path/to/nginx.conf
```

**配置文件语法检查**

```shell
nginx -t -c /path/to/nginx.conf
```

**nginx 进程控制**

```shell
nginx -s reload|reopen|stop|quit  #重新加载配置|重启|停止|退出 nginx
```

### nginx 详细配置

```yaml
user nobody;                                    # 运行用户
worker_processes  1;                            # 启动进程数量,通常设置成和cpu的数量相等

#worker_rlimit_nofile 65535;                    # nginx 最大打开文件数
# 全局错误日志及PID文件
error_log  /var/log/nginx/error.log;           # 可配置错误log的输出等级  notice|info 。。。
pid        /var/run/nginx.pid;

# 工作模式及连接数上限
events {
    accept_mutex on;                  #设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;                  #设置一个进程是否同时接受多个网络连接，默认为off
    use epoll;                        #事件驱动模型select|poll|kqueue|epoll|resig|/dev/poll|eventport
    worker_connections  1024;         #最大连接数，默认为512
}

# 设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
    include       mime.types;        # 设定mime类型,类型由mime.type文件定义  文件扩展名与文件类型映射表
    default_type  application/octet-stream;
    
    #设定日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log   /var/log/nginx/access.log  main;   # 服务日志 
    sendfile     on;          #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块
    #tcp_nopush     on;       # 按照大小发送数据包,不是按照请求
    keepalive_timeout  65;    # 连接超时时间
    tcp_nodelay     on;       # 按照请求发送数据包
    gzip  on;                 # 开启gzip压缩
    gzip_disable "MSIE [1-6].";   # 不给IE6 Gzip
    #gzip_min_length 1024;     # 启用压缩上限
    #gzip_buffers 416k;        # 缓存空间大小
    #gzip_comp_level 2;        # 定义压缩等级，默认为6，压缩比越大，效率越低
    #gzip_types text/plain application/x-javascript application/xml text/css;    #压缩文件类型
    #gzip_vary on;             #启用压缩标识
    #gzip_static on;           # 静态压缩
    client_header_buffer_size    128k;  # 如果请求头大小大于指定的缓冲区，则使用large_client_header_buffers指令分配更大的缓冲区。 
    large_client_header_buffers  4 128k; # 规定了用于读取大型客户端请求头的缓冲区的最大数量和大小。 这些缓冲区仅在缺省缓冲区不足时按需分配。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。

    upstream mysvr {          # 服务集群
      server xxx:xxx:xx:xxx:xxxx ;      # 可以给单个server配置weight权重
      server xxx.xxx.xx.xxx:xxxx backup;  #热备
    }
        
    #设定虚拟主机配置
    server {
        listen    80;                # 侦听80端口
        server_name  mysvr;          # 定义使用 mysvr(HTTP请求的Host字段) 访问
        root html;                   # 定义服务器的默认网站根目录位置
        access_log  logs/nginx.access.log  main;  # 设定本虚拟主机的访问日志
        
        # 配置路径转发
        location / {                                 # 默认请求
        	proxy_pass mysvr                         # 配置反向代理
            index index.php index.html index.htm;    # 定义首页索引文件的名称
            client_max_body_size 100m;               #  配置请求体最大容量
            proxy_set_header Host $http_host;            # Host表示客户端请求的HOST主机名称；
            proxy_set_header X-Real-IP $remote_addr;     # X-Real-IP表示客户端真实的IP；
            proxy_set_header REMOTE-HOST $remote_addr;   
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;     # X-Forwarded-For这个Header和X-Real-IP类似，但它在多层代理时会包含真实客户端及中间每个代理服务器的IP。
            proxy_set_header X-Forwarded-Proto $scheme;    # X-Forwarded-Proto表示客户端真实的协议（http还是https) 
            deny  127.0.0.1;              #拒绝的ip
            allow 172.18.5.54;            #允许的ip
            
            autoindex off;                 #激活/关闭自动索引
            autoindex_exact_size off;      #设定索引时文件大小的单位(B,KB, MB 或 GB)
            autoindex_localtime off;       #开启以本地时间来显示文件时间的功能。默认为关（GMT时间）
        }
        
        # 定义错误提示页面
        error_page   500 502 503 504 /50x.html;
        location = /50x.html {
        	root /root;
        }
        
        #静态文件，nginx自己处理
        location ~ ^/(images|javascript|js|css|flash|media|static)/ {
            expires 30d;   # 静态文件有效时间
        }
        
        #设定查看Nginx状态的地址
        location /NginxStatus {
            stub_status            on;
            access_log              on;
            auth_basic              "NginxStatus";
            auth_basic_user_file  conf/htpasswd;
        }
        
        #禁止访问 .htxxx 文件
        location ~ /.ht {
            deny all;
        }
    }
}
```

### Nginx 优化配置项

#### 进程数优化

nginx运行工作进程个数，一般设置cpu的核心或者核心数x2

查看cpu核数可以使用TOP命令后按1

**优化步骤:**

```shell
vim /usr/local/ninx/conf/nginx.conf
# 修改 worker_provesses 4
/usr/local/nginx/sbin/nginx -s reload
```

worker_processes最多开启8个，8个以上性能提升不会再提升了，而且稳定性变得更低，所以8个进程够用了。

#### 连接数量优化

**文件符限制配置**

Nginx最多可以打开文件句柄数，通过配置变量 worker_rlimit_nofile 65535; 来控制

这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n的值保持一致。

文件资源限制的配置可以在/etc/security/limits.conf设置，针对root/user等各个用户或者*代表所有用户来设置。

nginx默认采用epoll事件模型，处理效率高

work_connections是单个worker进程允许客户端最大连接数，这个数值一般根据服务器性能和内存来制定，实际最大值就是worker进程数乘以work_connections

实际我们填入一个65535，足够了，这些都算并发值，一个网站的并发达到这么大的数量，也算一个大站了！

**连接接收策略配置**

multi_accept 告诉nginx收到一个新连接通知后接受尽可能多的连接，默认是on，设置为on后，多个worker按串行方式来处理连接，也就是一个连接只有一个worker被唤醒，其他的处于休眠状态，设置为off后，多个worker按并行方式来处理连接，也就是一个连接会唤醒所有的worker，直到连接分配完毕，没有取得连接的继续休眠。当你的服务器连接数不多时，开启这个参数会让负载有一定的降低，但是当服务器的吞吐量很大时，为了效率，可以关闭这个参数。

**优化步骤:**

```shell
vim /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
* soft noproc 65535
* hard noproc 65535
# 重新登录用户
vim /usr/local/ninx/conf/nginx.conf
# 修改 user epoll
# worker_connections 65535;
# multi_accept off;
/usr/local/nginx/sbin/nginx -s reload
```

#### 配置高效传输模式

Include mime.types; //媒体类型,include 只是一个在当前文件中包含另一个文件内容的指令

default_type application/octet-stream; //默认媒体类型足够

sendfile on；//开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。

注意：如果图片和文件显示不正常把这个改成off。

tcp_nopush on；必须在sendfile开启模式才有效，防止网路阻塞，积极的减少网络报文段的数量（将响应头和正文的开始部分一起发送，而不一个接一个的发送。）

**优化步骤:**

```shell
vim /usr/local/ninx/conf/nginx.conf
#include mime.types;
#default_type application/octet-stream;
#sendfile on;
#tcp_nopush on;
/usr/local/nginx/sbin/nginx -s reload
```

#### 服务器资源占用优化

主要目的是保护服务器资源，CPU，内存，控制连接数，因为建立连接也是需要消耗资源的

keepalived_timeout客户端连接保持会话超时时间，超过这个时间，服务器断开这个链接

tcp_nodelay；也是防止网络阻塞，不过要包涵在keepalived参数才有效

client_header_buffer_size 4k;

客户端请求头部的缓冲区大小，这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过 1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE取得。

open_file_cache max=102400 inactive=20s;

这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件

数一致，inactive 是指经过多长时间文件没被请求后删除缓存。

open_file_cache_valid 30s;

这个是指多长时间检查一次缓存的有效信息。

open_file_cache_min_uses 1;

open_file_cache指令中的inactive 参数时间内文件的最少使用次数，如果超过这个数字，文

件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive 时间内一次没被使用，它将被移除。

client_header_timeout设置请求头的超时时间。我们也可以把这个设置低些，如果超过这个时间没有发送任何数据，nginx将返回request time out的错误

client_body_timeout设置请求体的超时时间。我们也可以把这个设置低些，超过这个时间没有发送任何数据，和上面一样的错误提示

reset_timeout_connection 告诉nginx关闭不响应的客户端连接。这将会释放那个客户端所占有的内存空间。

send_timeout响应客户端超时时间，这个超时时间仅限于两个活动之间的时间，如果超过这个时间，客户端没有任何活动，nginx关闭连接

server_tokens 并不会让nginx执行的速度更快，但它可以关闭在错误页面中的nginx版本数字，这样对于安全性是有好处的。

client_max_body_size上传文件大小限制

```shell
vim /usr/local/ninx/conf/nginx.conf
#keepalive_timeout 60;
#tcp_nodelay on;
#client_header_buffer_size 4k;
#open_file_cache max=102400 inactive=20s;
#open_file_cache_valid 30s;
#open_file_cache_min_uses 1;
#client_header_timeout 15;
#client_body_timeout 15;
#reset_timedout_connection on;
#send_timeout 15;
#server_tokens off;
#client_max_body_size 10m
/usr/local/nginx/sbin/nginx -s reload
```

#### PHP请求调优

fastcgi_connect_timeout 600; #指定连接到后端FastCGI的超时时间。

fastcgi_send_timeout 600; #向FastCGI传送请求的超时时间。

fastcgi_read_timeout 600; #指定接收FastCGI应答的超时时间。

fastcgi_buffer_size 64k; #指定读取FastCGI应答第一部分需要用多大的缓冲区，默认的缓冲区大小为fastcgi_buffers指令中的每块大小，可以将这个值设置更小。

fastcgi_buffers 4 64k; #指定本地需要用多少和多大的缓冲区来缓冲FastCGI的应答请求，如果一个php脚本所产生的页面大小为256KB，那么会分配4个64KB的缓冲区来缓存，如果页面大小大于256KB，那么大于256KB的部分会缓存到fastcgi_temp_path指定的路径中，但是这并不是好方法，因为内存中的数据处理速度要快于磁盘。一般这个值应该为站点中php脚本所产生的页面大小的中间值，如果站点大部分脚本所产生的页面大小为256KB，那么可以把这个值设置为“8 32K”、“4 64k”等。

fastcgi_busy_buffers_size 128k; #建议设置为fastcgi_buffers的两倍，繁忙时候的buffer

fastcgi_temp_file_write_size 128k; #在写入fastcgi_temp_path时将用多大的数据块，默认值是fastcgi_buffers的两倍，该数值设置小时若负载上来时可能报502BadGateway

fastcgi_temp_path #缓存临时目录

fastcgi_intercept_errors on;#这个指令指定是否传递4xx和5xx错误信息到客户端，或者允许nginx使用error_page处理错误信息。

注：静态文件不存在会返回404页面，但是php页面则返回空白页！！

fastcgi_cache_path /usr/local/nginx1.10/fastcgi_cachelevels=1:2 keys_zone=cache_fastcgi:128minactive=1d max_size=10g;# fastcgi_cache缓存目录，可以设置目录层级，比如1:2会生成16*256个子目录，cache_fastcgi是这个缓存空间的名字，cache是用多少内存（这样热门的内容nginx直接放内存，提高访问速度），inactive表示默认失效时间，如果缓存数据在失效时间内没有被访问,将被删除，max_size表示最多用多少硬盘空间。

fastcgi_cache cache_fastcgi; #表示开启FastCGI缓存并为其指定一个名称。开启缓存非常有用，可以有效降低CPU的负载，并且防止502的错误放生。cache_fastcgi为proxy_cache_path指令创建的缓存区名称

fastcgi_cache_valid 200 302 1h; #用来指定应答代码的缓存时间，实例中的值表示将200和302应答缓存一小时，要和fastcgi_cache配合使用

fastcgi_cache_valid 301 1d; #将301应答缓存一天

fastcgi_cache_valid any 1m; #将其他应答缓存为1分钟

fastcgi_cache_min_uses 1; #该指令用于设置经过多少次请求的相同URL将被缓存。

fastcgi_cache_key http://\$host\$request_uri; #该指令用来设置web缓存的Key值,nginx根据Key值md5哈希存储.一般根据\$host(域名)、\$request_uri(请求的路径)等变量组合成proxy_cache_key 。

fastcgi_pass #指定FastCGI服务器监听端口与地址，可以是本机或者其它

总结：

nginx的缓存功能有：proxy_cache / fastcgi_cache

proxy_cache的作用是缓存后端服务器的内容，可能是任何内容，包括静态的和动态。

fastcgi_cache的作用是缓存fastcgi生成的内容，很多情况是php生成的动态的内容。

proxy_cache缓存减少了nginx与后端通信的次数，节省了传输时间和后端宽带。

fastcgi_cache缓存减少了nginx与php的通信的次数，更减轻了php和数据库(mysql)的压力。

**优化方式**

```shell
fastcgi_connect_timeout 600;
fastcgi_send_timeout 600;
fastcgi_read_timeout 600;
fastcgi_buffer_size 64k;
fastcgi_buffers 4 64k;
fastcgi_busy_buffers_size 128k;
fastcgi_temp_file_write_size 128k;
fastcgi_temp_path/usr/local/nginx1.10/nginx_tmp;
fastcgi_intercept_errors on;
fastcgi_cache_path/usr/local/nginx1.10/fastcgi_cache levels=1:2 keys_zone=cache_fastcgi:128minactive=1d max_size=10g;
```

#### gzip 数据压缩调优

使用gzip压缩功能，可能为我们节约带宽，加快传输速度，有更好的体验，也为我们节约成本，所以说这是一个重点。

Nginx启用压缩功能需要你来ngx_http_gzip_module模块，apache使用的是mod_deflate

一般我们需要压缩的内容有：文本，js，html，css，对于图片，视频，flash什么的不压缩，同时也要注意，我们使用gzip的功能是需要消耗CPU的！

gzip on; #开启压缩功能

gzip_min_length 1k; #设置允许压缩的页面最小字节数，页面字节数从header头的Content-Length中获取，默认值是0，不管页面多大都进行压缩，建议设置成大于1K，如果小与1K可能会越压越大。

gzip_buffers 4 32k; #压缩缓冲区大小，表示申请4个单位为32K的内存作为压缩结果流缓存，默认值是申请与原始数据大小相同的内存空间来存储gzip压缩结果。

gzip_http_version 1.1; #压缩版本，用于设置识别HTTP协议版本，默认是1.1，目前大部分浏览器已经支持GZIP解压，使用默认即可

gzip_comp_level 6; #压缩比例，用来指定GZIP压缩比，1压缩比最小，处理速度最快，9压缩比最大，传输速度快，但是处理慢，也比较消耗CPU资源。

gzip_types text/css text/xml application/javascript; #用来指定压缩的类型，‘text/html’类型总是会被压缩。

默认值: gzip_types text/html (默认不对js/css文件进行压缩)

\# 压缩类型，匹配MIME型进行压缩

\# 不能用通配符 text/*

\# (无论是否指定)text/html默认已经压缩

\# 设置哪压缩种文本文件可参考 conf/mime.types

gzip_vary on; #varyheader支持，改选项可以让前端的缓存服务器缓存经过GZIP压缩的页面，例如用Squid缓存经过nginx压缩的数据

**优化方式**

```shell
gzip on;

gzip_min_length 2k;

gzip_buffers 4 32k;

gzip_http_version 1.1;

gzip_comp_level 6;

gzip_typestext/plain text/css text/javascriptapplication/json application/javascript application/x-javascriptapplication/xml;

gzip_vary on;

gzip_proxied any;
```

#### 使用缓存来优化效率

缓存，主要针对于图片，css，js等元素更改机会比较少的情况下使用，特别是图片，占用带宽大，我们完全可以设置图片在浏览器本地缓存365d，css，js，html可以缓存个10来天，这样用户第一次打开加载慢一点，第二次，就非常快了！缓存的时候，我们需要将需要缓存的拓展名列出来， Expires缓存配置在server字段里面

**配置方式**

```shell
location  ~ * \.(ico|jpe?g|gif|png|bmp|swf|flv)$ {

expires 30d;

\#log_not_found off;

access_log off;

}

location ~ * \.(js|css)$ {

expires 7d;

log_not_found off;

access_log off;

}
```

注：log_not_found off;是否在error_log中记录不存在的错误。默认是。

**expire功能优点** 

（1）expires可以降低网站购买的带宽，节约成本

（2）同时提升用户访问体验

（3）减轻服务的压力，节约服务器成本，是web服务非常重要的功能。

**expire功能缺点**

被缓存的页面或数据更新了，用户看到的可能还是旧的内容，反而影响用户体验。解决办法：第一个缩短缓存时间，例如：1天，但不彻底，除非更新频率大于1天；第二个对缓存的对象改名。

**网站不希望被缓存的内容** 

1）网站流量统计工具

2）更新频繁的文件（google的logo）

#### 使用防盗链

防止别人直接从你网站引用图片等链接，消耗了你的资源和网络流量，那么我们的解决办法由几种：

 1：水印，品牌宣传，你的带宽，服务器足够

 2：防火墙，直接控制，前提是你知道IP来源 

 3：防盗链策略下面的方法是直接给予404的错误提示

**优化方法**

```shell
location ~ *^.+\.(jpg|gif|png|swf|flv|wma|wmv|asf|mp3|mmf|zip|rar)$ {
valid_referers noneblocked www.benet.com benet.com;
if($invalid_referer) {
    #return 302 http://www.benet.com/img/nolink.jpg;
	return 404;
	break;
}
	access_log off;
}
```

参数可以使如下形式：

none 意思是不存在的Referer头(表示空的，也就是直接访问，比如直接在浏览器打开一个图片)

blocked 意为根据防火墙伪装Referer头，如：“Referer:XXXXXXX”。

server_names 为一个或多个服务器的列表，0.5.33版本以后可以在名称中使用“*”通配符。

#### 内核参数优化

fs.file-max = 999999：                      这个参数表示进程（比如一个worker进程）可以同时打开的最大句柄数，这个参数直线限制最大并发连接数，需根据实际情况配置。

net.ipv4.tcp_max_tw_buckets = 6000                          #这个参数表示操作系统允许TIME_WAIT套接字数量的最大值，如果超过这个数字，TIME_WAIT套接字将立刻被清除并打印警告信息。该参数默认为180000，过多的TIME_WAIT套接字会使Web服务器变慢。

注：主动关闭连接的服务端会产生TIME_WAIT状态的连接

net.ipv4.ip_local_port_range = 1024 65000                      #允许系统打开的端口范围。

net.ipv4.tcp_tw_recycle = 1                                                #启用timewait快速回收。

net.ipv4.tcp_tw_reuse = 1                                                  #开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接。这对于服务器来说很有意义，因为服务器上总会有大量TIME-WAIT状态的连接。

net.ipv4.tcp_keepalive_time = 3                                         # 这个参数表示当keepalive启用时，TCP发送keepalive消息的频度。默认是2小时，若将其设置的小一些，可以更快地清理无效的连接。

net.ipv4.tcp_syncookies = 1                                   #开启SYN Cookies，当出现SYN等待队列溢出时，启用cookies来处理。

net.core.somaxconn = 40960                                         #web 应用中 listen 函数的 backlog 默认会给我们内核参数的 net.core.somaxconn 限制到128，而nginx定义的NGX_LISTEN_BACKLOG 默认为511，所以有必要调整这个值。

注：对于一个TCP连接，Server与Client需要通过三次握手来建立网络连接.当三次握手成功后,我们可以看到端口的状态由LISTEN转变为ESTABLISHED,接着这条链路上就可以开始传送数据了.每一个处于监听(Listen)状态的端口,都有自己的监听队列.监听队列的长度与如somaxconn参数和使用该端口的程序中listen()函数有关

somaxconn定义                               # 了系统中每一个端口最大的监听队列的长度,这是个全局的参数,默认值为128，对于一个经常处理新连接的高负载 web服务环境来说，默认的 128 太小了。大多数环境这个值建议增加到 1024 或者更多。大的侦听队列对防止拒绝服务 DoS 攻击也会有所帮助。

net.core.netdev_max_backlog = 262144                       #每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

net.ipv4.tcp_max_syn_backlog = 262144                          #这个参数标示TCP三次握手建立阶段接受SYN请求队列的最大长度，默认为1024，将其设置得大一些可以使出现Nginx繁忙来不及accept新连接的情况时，Linux不至于丢失客户端发起的连接请求。

net.ipv4.tcp_rmem = 10240 87380 12582912                              #这个参数定义了TCP接受缓存（用于TCP接受滑动窗口）的最小值、默认值、最大值。

net.ipv4.tcp_wmem = 10240 87380 1258291： #                         这个参数定义了TCP发送缓存（用于TCP发送滑动窗口）的最小值、默认值、最大值。

net.core.rmem_default = 6291456                #这个参数表示内核套接字接受缓存区默认的大小。

net.core.wmem_default = 6291456               #这个参数表示内核套接字发送缓存区默认的大小。

net.core.rmem_max = 12582912                 # 这个参数表示内核套接字接受缓存区的最大大小。

net.core.wmem_max = 12582912                #  这个参数表示内核套接字发送缓存区的最大大小。

net.ipv4.tcp_syncookies = 1                         # 该参数与性能无关，用于解决TCP的SYN攻击。

**贴一个完整的内核优化设置**

```shell
fs.file-max = 999999

net.ipv4.ip_forward = 0

net.ipv4.conf.default.rp_filter = 1

net.ipv4.conf.default.accept_source_route = 0

kernel.sysrq = 0

kernel.core_uses_pid = 1

net.ipv4.tcp_syncookies = 1

kernel.msgmnb = 65536

kernel.msgmax = 65536

kernel.shmmax = 68719476736

kernel.shmall = 4294967296

net.ipv4.tcp_max_tw_buckets = 6000

net.ipv4.tcp_sack = 1

net.ipv4.tcp_window_scaling = 1

net.ipv4.tcp_rmem = 10240 87380 12582912

net.ipv4.tcp_wmem = 10240 87380 12582912

net.core.wmem_default = 8388608

net.core.rmem_default = 8388608

net.core.rmem_max = 16777216

net.core.wmem_max = 16777216

net.core.netdev_max_backlog = 262144

net.core.somaxconn = 40960

net.ipv4.tcp_max_orphans = 3276800

net.ipv4.tcp_max_syn_backlog = 262144

net.ipv4.tcp_timestamps = 0

net.ipv4.tcp_synack_retries = 1

net.ipv4.tcp_syn_retries = 1

net.ipv4.tcp_tw_recycle = 1

net.ipv4.tcp_tw_reuse = 1

net.ipv4.tcp_mem = 94500000 915000000 927000000

net.ipv4.tcp_fin_timeout = 1

net.ipv4.tcp_keepalive_time = 30

net.ipv4.ip_local_port_range = 1024 65000

执行sysctl -p  使内核修改生效果
```