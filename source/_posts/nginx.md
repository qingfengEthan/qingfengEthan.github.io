---
title: 使用nginx做负载均衡
date: 2018-11-30 16:35:21
tags:
---

------

&ensp;&ensp;&ensp;&ensp;Nginx是一款自由的、开源的、高性能的HTTP服务器和反向代理服务器；同时也是一个IMAP、POP3、SMTP代理服务器；Nginx可以作为一个HTTP服务器进行网站的发布处理，另外nginx可以作为反向代理进行负载均衡的实现。Nginx比Apache更轻量级、并发性能更好。

> * Nginx使用基于事件驱动架构，使得其可以支持数以百万级别的TCP连接（Apache并发达到数万时，性能就会急剧下降）
> * 高度的模块化和自由软件许可证是的第三方模块层出不穷
> * Nginx是一个跨平台服务器，可以运行在Linux, FreeBSD, Solaris, AIX, Mac OS, Windows等操作系统上

### 正向代理和反向代理
&ensp;&ensp;&ensp;&ensp;**正向代理**：正向代理是一个位于客户端和目标服务器之间的代理服务器为了从原始服务器取得内容，客户端向代理服务器发送一个请求，并且指定目标服务器，之后代理向目标服务器转交并且将获得的内容返回给客户端。客户端需要设置后才可以访问，正向代理模式屏蔽或者隐藏了真实客户端信息。
![cmd-markdown-logo](http://139.224.113.197/20181128142716.png)
&ensp;&ensp;&ensp;&ensp;**反向代理**：对于客户端来说，反向代理就像目标服务器一样，并且客户端不需要进行任何设置。客户端向反向代理发送请求，接着反向代理判断请求走向何处，并将请求转交给客户端，客户端并不会感知到反向代理后面的服务，也因此不需要客户端做任何设置。

![cmd-markdown-logo](http://139.224.113.197/20181128142734.png)

主要区别：

 - [ ] 是否指定目标服务器
 - [ ] 客户端是否要做设置

**实际应用场景**

![cmd-markdown-logo](http://139.224.113.197/20181128142751.png)

### nginx工作原理
![cmd-markdown-logo](http://139.224.113.197/20181128142914.png)

&ensp;&ensp;&ensp;&ensp;nginx是以多进程的方式来工作，启动后会有一个master进程和多个worker进程。
master进程：
1、接收来自外界的信号，向各worker进程发送信号。
2、监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程。
work进程：
由master进程生成，每个进程的socket会监控在同一个ip地址与端口，当一个连接进来后，所有在accept在这个socket上面的进程，都会收到通知，同时只有一个进程可以accept这个连接，然后开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后断开连接。


### nginx模块配置
#### 1. nginx配置说明


| 配置区域        | 说明   |
| --------   | -------- |
| main块     | 配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。 |
| events块       |   配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。   |
| http块       |    可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等    |
| upstream块        |    配置HTTP负载均衡器分配流量到几个应用程序服务器。    |
| server块        |    配置虚拟主机的相关参数，一个http中可以有多个server。    |
| location块       |    配置请求的路由，以及允许根据用户请求的URI来匹配指定的各location以进行访问配置；匹配到时，将被location块中的配置所处理。    |

nginx文件结构如下：
```bash
... #main全局块  
#events块   
events {  
...  
}  
#http块   
http  
{  
... #http全局块 
    # upstream负载均衡块 
    upstream … # upstream负载均衡块  
    {  
     …  
    }  
    #server块  
    server 
    {  
    ... #server全局块 
    
        #location块 
        location [PATTERN]  
        {  
            ...  
        }  
        location [PATTERN]  
        { 
            ...  
        }  
    }  
    server  
    { 
        ...  
    }  
... #http全局块  
}
```
#### 2. 配置实例

```bash
#nginx用户及组： 用户 所在组
user  root root;
#工作进程数，通常等于CPU数量或者2倍于CPU 
worker_processes 8;
#开启利用多核cpu配置
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
# 指定nginx进程运行文件存放地址 
pid  logs/nginx.pid;
# [ debug | info | notice | warn | error | crit ]
#error_log
error_log  /Data/logs/nginx/nginx_error.log info;
#worker进程的最大打开文件数限制
worker_rlimit_nofile 51200;
 
events
{
      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport，linux 2.6以上应使用epoll
       use epoll;   
 
       #maxclient = worker_processes * worker_connections / cpu_number,一个worker进程同时打开的最大连接数,可设成"auto"
       worker_connections 51200;
}
 
http
{
       #文件扩展名与文件类型映射表
       include        mime.types;
       #默认文件类型
       default_type  application/octet-stream;
       #charset;
       charset utf-8;
       #自定义日志格式
       log_format logFormat '$remote_addr|$http_x_forwarded_for|[$time_local]|$http_host|$request|$status|$body_bytes_sent|$request_time|$upstream_response_time|$upstream_cache_status|$http_referer|$http_user_agent|$upstream_addr|$real_ip';

       # resolver: local dns servers
       resolver     10.66.0.10     10.66.0.11   ;
          
       #General Options
       server_names_hash_bucket_size 128;
       client_header_buffer_size 512k;
       large_client_header_buffers 4 512k;
       client_body_buffer_size    8m; #256k 
   
       #server_tokens off;
       ignore_invalid_headers   on;
       recursive_error_pages    on;
       server_name_in_redirect off;
      
       #允许sendfile方式传输文件
       sendfile                 on;
 
       #连接超时时间
       keepalive_timeout 80s;
       keepalive_requests 10000;

       #TCP Options 
       tcp_nopush  on;
       tcp_nodelay on;

       #fastcgi options 外部动态程序的直接调用或者解析 
       fastcgi_connect_timeout 300;
       fastcgi_send_timeout 300;
       fastcgi_read_timeout 300;
       fastcgi_buffer_size 64k;
       fastcgi_buffers 4 64k;
       fastcgi_busy_buffers_size 128k;
       fastcgi_temp_file_write_size 128k;
    
       #size limits
       client_max_body_size       50m;

       gzip on;
       gzip_min_length  1k;
       gzip_buffers     4 16k;
       #gzip_http_version 1.0;
       gzip_comp_level 2;
       gzip_types       text/plain application/x-javascript text/css application/xml application/json;
       gzip_vary on; 

        fastcgi_temp_path          /dev/shm/fastcgi_temp;
        client_body_temp_path      /dev/shm/client_body_temp; 
     
      #fore gateway
      upstream foreGw {
           server 10.66.0.13:8080 max_fails=10 fail_timeout=5s;
           server 10.66.0.15:8080 max_fails=10 fail_timeout=5s;
           keepalive 100;
      }
    
     # platform gateway
     upstream platformGw {
              server 10.66.10.21:8089 max_fails=10 fail_timeout=5s;
              server 10.66.10.22:8089 max_fails=10  fail_timeout=5s;
       keepalive 100;
     }
       
       include          vhosts/api.fore.cn.conf;
       include          vhosts/api.platform.cn.conf;       
 }
```

#### 3. upstream负载均衡块
Nginx支持的负载均衡调度算法方式如下：
> * 轮询（默认）：每一个请求按时间顺序逐一分配到不同的后端服务器。假设后端服务器down掉。能自己主动剔除。
```bash
 upstream erpgateway {
    server 10.66.0.112:8089;
    server 10.66.0.114:8089;
 }
```

> * weight加权：接收到的请求按照顺序逐一分配到不同的后端服务器，即使在使用过程中，某一台后端服务器宕机，Nginx会自动将该服务器剔除出队列，请求受理情况不会受到任何影响。 这种方式下，可以给不同的后端服务器设置一个权重值（weight），用于调整不同的服务器上请求的分配率；权重数据越大，被分配到请求的几率越大；该权重值，主要是针对实际工作环境中不同的后端服务器硬件配置进行调整的。
```bash
 upstream gateway {
    server 10.66.0.112:8089 weight=5;
    server 10.66.0.114:8089 weight=10;
 }
```
> * ip_hash：每个请求按照发起客户端的ip的hash结果进行匹配，这样的算法下一个固定ip地址的客户端总会访问到同一个后端服务器，这也在一定程度上解决了集群部署环境下session共享的问题。
```bash
 upstream gateway {
     ip_hash;
    server 10.66.0.112:8089;
    server 10.66.0.114:8089;
 }
```
> * fair（第三方）：智能调整调度算法，动态的根据后端服务器的请求处理到响应的时间进行均衡分配，响应时间短处理效率高的服务器分配到请求的概率高，响应时间长处理效率低的服务器分配到的请求少；结合了前两者的优点的一种调度算法。Nginx默认不支持fair算法，如果要使用这种调度算法，请安装upstream_fair模块。
```bash
 upstream gateway {
    server 10.66.0.112:8089;
    server 10.66.0.114:8089;
    fair;
 }
```
> * url_hash（第三方）：按照访问的url的hash结果分配请求，每个请求的url会指向后端固定的某个服务器，可以在nginx作为静态服务器的情况下提高缓存效率。同样要注意Nginx默认不支持这种调度算法，要使用的话需要安装nginx的hash软件包。
```bash
 upstream gateway {
    server 10.66.0.112:8089;
    server 10.66.0.114:8089;
    hash $request_uri; 
    hash_method crc32; 
 }
```
定义负载均衡设备的Ip及设备状态
```bash
upstream bakend{
    ip_hash;
    server 10.66.0.111:8089 down;
    server 10.66.0.112:8089 weight=2;
    server 10.66.0.113:8089 max_fails=5  fail_timeout=3s;
    server 10.66.0.114:8089 backup;
}
```
max_fails=5  fail_timeout=3s代表在3秒内请求某一应用失败5次，认为该应用宕机，后等待3秒，这期间内不会再把新请求发送到宕机应用，而是直接发到正常的那一台，时间到后再有请求进来继续尝试连接宕机应用且仅尝试1次，如果还是失败，则继续等待3秒...以此循环，直到恢复。

#### 4. location路由模块

location [=|~|~*|^~] /uri/ { … }

| 模式       | 含义   | 
| --------   | -----  | 
| location = /uri     | = 表示精确匹配，只有完全匹配上才能生效 | 
| location ^~ /uri        |   ^~ 开头对URL路径进行前缀匹配，并且在正则之前。  |  
| location ~ pattern        |    开头表示区分大小写的正则匹配    | 
| location ~* pattern        |    开头表示不区分大小写的正则匹配   | 
| location /uri       |    不带任何修饰符，也表示前缀匹配，但是在正则匹配之后    | 
| location /        |    通用匹配，任何未匹配到其它location的请求都会匹配到，相当于switch中的default    | 

多个 location 配置的情况下匹配顺序为:
> * 首先精确匹配 =
> * 其次前缀匹配 ^~
> * 其次是按文件中顺序的正则匹配
> * 然后匹配不带任何修饰的前缀匹配。
> * 最后是交给 / 通用匹配
> * 当有匹配成功时候，停止匹配，按当前匹配规则处理请求

参数proxy_redirect将被代理服务器的响应头中的location/refresh字段进行修改后返回给客户端

#### 5. nginx内置参数
| 参数名        | 功能   | 
| ------ | --------  |
|$arg_parameter|如果在请求中设置了查询字符串，那么这个变量包含在查询字符串是GET请求PARAMETER中的值  |
|   $args   | 该变量的值是GET请求在请求行中的参数。 |
|   $binary_remote_addr   | 二进制格式的客户端地址 |
|   $body_bytes_sent   | 响应体的大小，即使发生了中断或者是放弃，也是一样的准确 |
|   $cookie_COOKIE   | 该变量的值是cookie COOKIE的值 |
|   $document_root   | 该变量的值为当前请求的location（http，server，location，location中的if）中root指令中指定的值 |
|   $document_uri   | 同$uri |
|   $host   | 该变量的值等于请求头中Host的值。如果Host无效时，那么就是处理该请求的server的名称 |
|   $hostname   | 有gethostname返回值设置机器名 |
|   $http_HEADER   | 该变量的值为HTTP 请求头HEADER，具体使用时会转换为小写，并且将“——”（破折号）转换为"_"(下划线) |
|   $is_args   | 如果设置了\$args，那么值为“？”，否则为“” |
|   $limit_rate  | 该变量允许限制连接速率 |
|   $nginx_version   | 当前运行的nginx的版本号 |
|   $query_string  | 同$args |
|   $remote_addr   | 客户端的IP地址 |
|   $remote_user   | 该变量等于用户的名字，基本身份验证模块使用 |
|   $remote_port  | 客户端连接端口 |
|   $request_filename   | 该变量等于当前请求文件的路径，有指令root或者alias和URI构成 |
|   $request_body| 该变量包含了请求体的主要信息。该变量与proxy_pass或者fastcgi_pass相关 |
|   $request_body_file    | 客户端请求体的临时文件 |
|   $request_completion   |如果请求成功完成，那么显示“OK”。如果请求没有完成或者请求不是该请求系列的最后一部分，那么它的值为空  |
|   $request_method    | 该变量的值通常是GET或者POST |
|   $request_uri   | 该变量的值等于原始的URI请求，就是说从客户端收到的参数包括了原始请求的URI，该值是不可以被修改的，不包含主机名，例如“/foo/bar.php?arg=baz” |
|   $scheme    |该变量表示HTTP scheme（例如HTTP，HTTPS），根据实际使用情况来决定  |
|   $server_addr   | 该变量的值等于服务器的地址。通常来说，在完成一次系统调用之后就会获取变量的值，为了避开系统钓鱼，那么必须在listen指令中使用bind参数 |
|   $server_name    | 该变量为server的名字 |
|   $server)port   | 该变量等于接收请求的端口 |
|   $server_protocol    | 该变量的值为请求协议的值，通常是HTTP/1.0或者HTTP/1.1 |
|   $uri    | 该变量的值等于当前请求中的URI（没有参数，不包括\$args）的值。它的值不同于request_uri，由浏览器客户端发送的request_uri的值。例如，可能会被内部重定向或者使用index,另外需要注意：$uri不包含主机名，例如 "/foo/bar.html" |





