
## 主模块


```
# 配置用户或者组，默认为nobody nobody。
user www www;  

 #Nginx开启的worker进程数，建议为CPU的核数
worker_processes 2; 

#指定nginx进程运行文件存放地址
pid /nginx/pid/nginx.pid;

#指定日志路径，级别。这个设置可以放入全局块、http块、server块，级别以此为：debug|info|notice|warn|error|crit|alert|emerg
error_log log/error.log debug; 

#可以在任意地方使用include指令实现配置文件的包含，类似于apache中的include方法，可减少主配置文件长度。
include vhosts/*.conf;
```


## 事件模块
```
events {
    #设置网路连接序列化，防止惊群现象发生，默认为on
    accept_mutex on; 
    
    #默认: 500ms 如果一个进程没有互斥锁，它将延迟至少多长时间。默认情况下，延迟是500ms 。
    accept_mutex_delay 100ms; 
    
    #设置一个进程是否同时接受多个网络连接，默认为off
    multi_accept on;
    
    #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport，不建议设置，nginx会自行选择
    use epoll;
    
    #最大连接数，默认为512
    worker_connections  1024;
}
```

## http部分

```
http {
    #文件扩展名与文件类型映射表
    include       mime.types;
    
    # 默认文件类型，默认为text/plain
    default_type  application/octet-stream; 
    
    #取消访问日志 
    access_log off; 
    
    #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
    sendfile on;   
    
    #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    sendfile_max_chunk 100k;  
    
    #连接超时时间，默认为75s，可以在http，server，location块。
    keepalive_timeout 65;  
    
    #开启gzip资源压缩
    gzip  on;  

    # 负载均衡
    upstream blog {   
        server 192.167.20.19:8081;
        server 192.168.10.121:8080 weight=5;
    }
    
    #设定请求缓冲
    client_header_buffer_size    128k;
    large_client_header_buffers  4 128k;
    
    #上传文件的大小限制  默认1m
    client_max_body_size 8m;

    server {
        #单连接请求上限次数。
        keepalive_requests 120;
        
        #监听端口
        listen       80;   
        
        #监听地址
        server_name  blog.13sai.com;  
        
        #自定义日志格式
        log_format  main '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
                      
        access_log  /data/logs/access.log  main;
        
        # 根目录
        root /www/web/public; 
        
        # 定义错误提示页面
        error_page   500 502 503 504 /50x.html;
        
        location /static/ {
            #root与alias主要区别在于nginx如何解释location后面的uri，这会使两者分别以不同的方式将请求映射到服务器文件上。
            #root的处理结果是：root路径＋location路径
            #alias的处理结果是：使用alias路径替换location路径
            alias /www/static/;
            
            #过期30天，静态文件不怎么更新，过期可以设大一点,如果频繁更新，则可以设置得小一点。
            expires 30d;
        }
        
        # 处理php请求到fpm端口
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
        
        location / {
            proxy_set_header Host $host;
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_pass  http://blog;  #请求转向blog 定义的服务器列表 
        }
        
        #禁止访问文件
        location ~ /.git {
            deny all;
            allow 127.0.0.1; #允许的ip 
        }
    }
} 
```
## 部分参数详细说明

#### server_name 

```
1.首先选择所有字符串完全匹配的server_name，如 blog.13sai.com 。
2.其次选择通配符在前面的server_name，如 *.13sai.com。
3.再次选择通配符在后面的server_name，如www.13sai.* 。 
4.最后选择使用正则表达式才匹配的server_name，如 ~^\.sai\.com$

如果都不匹配
1、优先选择listen配置项后有default或default_server的 
2、找到匹配listen端口的第一个server块
```

#### location
```
location
语法: location[=|~|~*|^~|@]/uri/{...}
配置块: server location会尝试根据用户请求中的URI来匹配上面的/uri表达式，如果可以匹配，就选择 location{}块中的配置来处理用户请求。
```

location表达式类型

```
~ 表示执行一个正则匹配，区分大小写;
~* 表示执行一个正则匹配，不区分大小写;
^~ 表示普通字符匹配。使用前缀匹配。如果匹配成功，则不再匹配其他location; 
= 进行普通字符精确匹配。也就是完全匹配;
@ 它定义一个命名的 location，使用在内部定向时，例如 error_page, try_files
```

优先级:

- 等号类型(=)的优先级最高。一旦匹配成功，则不再查找其他匹配项 
- 前缀普通匹配(^~)优先级次之。不支持正则表达式。使用前缀匹配，如果有多个location匹配的话，则使用表达式最长的那个 
- 正则表达式类型(~ ~*)的优先级次之。一旦匹配成功，则不再查找其他匹配项 
- 常规字符串匹配，如果有多个location匹配的话，则使用表达式最长的那个

> (location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径)

#### return 
```
语法:return code [text] return code URL;
return URL;
配置块:server，location，if
该指令用于结束规则的执行并返回状态吗给客户端。
状态码包括:
204(No Content)、
400(Bad Request)、
402(Payment Required)、
403(Forbidden) 
404(Not Found)、
405(Method Not Allowed)、
406(Not Acceptable)、 
408(Request Timeout)、
410(Gone)、
411(Length Required)、
413(Request Entity Too Large)、
416(Requested Range Not Satisfiable)、 
500(Internal Server Error)、
501(Not Implemented)、
502(Bad Gateway)、 
503(Service Unavailable)
504(Gateway Timeout)。

例如，示例，如果访问的URL以.sh .bash 结尾，返回状态码403 
location ~ .*\.(sh|bash)?$ {
	return 403;
}
```


#### rewrite

```
执行顺序：
1. 执行server块的rewrite指令(这里的块指的是server关键字后{}包围的区域，其它xx块类似)
2. 执行location匹配
3. 执行选定的location中的rewrite指令
如果其中某步URI被重写，则重新循环执行1-3，直到找到真实存在的文件
如果循环超过10次，则返回500 Internal Server Error错误

语法:rewrite regex replacement [flag]; 
默认值:—
配置块:server, location, if
rewrite是实现URL重写的关键指令，根据regex(正则表达式)部分内容，重定向到replacement，结尾是flag标记。 正则:perl兼容正则表达式语句进行规则匹配
替代内容:将正则匹配的内容替换成replacement
flag标记:rewrite支持的flag标记
```
###### if指令

```
语法：if(condition){...}
默认值：无
配置块：server,location
对给定的条件condition进行判断。如果为真，大括号内的rewrite指令将被执行。
if条件(conditon)可以是如下任何内容:

一个变量名；false如果这个变量是空字符串或者以0开始的字符串；
使用=,!= 比较的一个变量和字符串
是用~,~*与正则表达式匹配的变量，如果这个正则表达式中包含}，;则整个表达式需要用" 或' 包围
使用-f,!-f 检查一个文件是否存在
使用-d,!-d 检查一个目录是否存在
使用-e,!-e 检查一个文件、目录、符号链接是否存在
使用-x,!-x 检查一个文件是否可执行
```

###### if实例

```
if ($http_user_agent~*(mobile|nokia|iphone|ipad|android|samsung|htc|blackberry)) {
    rewrite ^.+ /mobile last; ＃跳转到手机站
}


if ($request_method = POST) {
    return 405;
}

if ($slow) {
    limit_rate 10k;
}

if ($invalid_referer) {
    return 403;
}
```

###### last & break

```
（1）last 和 break 当出现在location 之外时，两者的作用是一致的没有任何差异。需要注意的一点就是，他们会跳过所有的在他们之后的rewrite 模块中的指令，去选择自己匹配的location
（2）last 和 break 当出现在location 内部时，两者就存在了差异
-- last: 使用了last 指令，rewrite 后会跳出location 作用域，重新开始再走一次刚刚的行为
-- break: 使用了break 指令，rewrite后不会跳出location 作用域。它的生命也在这个location中终结。

解释通俗易懂：

last：重新将rewrite后的地址在server标签中执行
break：将rewrite后的地址在当前location标签中执行
```


######  permanent & redirect:
```
permanent: 永久性重定向。请求日志中的状态码为301
redirect:临时重定向。请求日志中的状态码为302
```

从实现功能的角度上去看，permanent 和 redirect 是一样的。不存在好坏。也不存在什么性能上的问题。但是对seo会有影响，这里要根据需要做出选择。

在 permanent 和 redirect  中提到了 状态码 301 和 302。 


记住：last 和 break 想对于的访问日志的请求状态码为200

当你打开一个网页，同时打开debug 模式时，会发现301 和 302 时的行为是这样的。

第一个请求301 或者 302 后，浏览器重新获取了一个新的URL ，然后会对这个新的URL 重新进行访问。所以当你配置的是permanent 和 redirect ,你对一个URL 的访问请求，落到服务器上至少为2次；而当你配置了last 或者是break 时，你最终的URL 确定下来后，不会将这个URL返回给浏览器，而是将其扔给了fastcgi_pass或者是proxy_pass指令去处理。请求一个URL ，落到服务器上的次数就为1次。


注意：配置last 在跨域的时候效果和redirect一致，都是返回302状态码，请求地址也发生改变



## 应用

#### 估算并发

nginx作为http服务器的时候：

    max_clients = worker_processes * worker_connections/2

nginx作为反向代理服务器的时候：

    max_clients = worker_processes * worker_connections/4  


#### 限制每个IP的并发连接数

定义一个叫“two”的记录区，总容量为 10M（超过大小将请求失败，以变量 $binary_remote_addr 作为会话的判断基准（即一个地址一个会话）。 限制 /download/ 目录下，一个会话只能进行一个连接。 简单点，就是限制 /download/ 目录下，一个IP只能发起一个连接，多过一个，一律503。

```
http {
    ...
    limit_conn_zone $binary_remote_addr zone=two:10m;

    server {
        ...
        
        location /download {
            limit_conn   two  1;
        }
    }
}
```


#### 限流

定义一个叫“one”的记录区，占用空间大小为10m（超过大小将请求失败），平均处理的请求频率不能超过每秒一次，也可以设置分钟速率

```
http {
    ...
    
    
    limit_req_zone  $binary_remote_addr  zone=one:10m  rate=1r/s;
    

    server {
        ...
        
        location / {
            #缓存区队列burst=5个,nodelay表示不延期(超过的请求失败)，即每秒最多可处理rate+burst个,同时处理rate个。
            limit_req zone=one burst=5 nodelay; 
        }
    }
}
```



#### 白名单

```
http{
    ...
    
    #判断客户端的ip地址是否在白名单列表当中,如果返回为0,则在白名单列表当中,否则返回为1
    geo $whiteIpList {
        default  1;
        118.24.109.254 0;
        47.98.147.0/24 1;
        #可以引入一些白名单配置
        include 'whiteIP.conf'
    }
    
    #如果不在白名单之内,返回客户端的二进制的ip地址
    map $whiteIpList  $limit {
        default  "";
        1   $binary_remote_addr;
        0   "";
    }
    
    #如果返回的是空字符串那么速率限制会失效
    limit_req_zone $limit zone=test:2m rate=1r/m;
    
    ...
}
```

#### 防盗链

```
http {
    ...

    server {
        ...
        
        location ~* \.(gif|jpg|png|swf|flv)$ {
            valid_referers none blocked *.13sai.com;
            if ($invalid_referer) {
                rewrite ^/ blog.13sai.com
            }
        }
    }
}
```

