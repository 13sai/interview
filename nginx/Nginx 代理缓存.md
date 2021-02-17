Nginx的缓存可以简单分成web缓存和代理缓存，本篇文章主要介绍代理缓存。

## web缓存

Nginx提供了expires、etag、if-modified-since指令来实现浏览器缓存控制。

这个配置比较简单，一般可以缓存一些js、css等静态文件。

对于这几个不想做过多说明，大家可以看两张图，简单理解下。

![浏览器缓存](https://cdn.learnku.com/uploads/images/201911/19/41489/ZEbgycTmrL.png!/fw/1240)

![Nginx 代理缓存](https://cdn.learnku.com/uploads/images/201911/20/41489/Vo6VBgbiUr.png!/fw/1240)


## 代理缓存

代理缓存主要用到proxy模块中的proxy_cache。我们来看一个demo。

```
upstream 13sai{
    server 127.0.0.1:9501 weight=10;
}


#自定义缓存目录,缓存文件大小
proxy_cache_path  /usr/local/etc/nginx/cache  levels=1:2  keys_zone=sai_cache:10m  max_size=200m inactive=10m  use_temp_path=off;

server {
    listen       80;
    server_name  nginx-t.com;

    location / {
        proxy_next_upstream error http_503;
        proxy_pass http://13sai;

        #启用缓存sai_cache
        proxy_cache sai_cache; 

        #定义如何生成缓存的键
        proxy_cache_key $scheme$proxy_host$uri$is_args$args;

        #针对多种请求方法缓存，默认GET HEAD
        proxy_cache_methods GET HEAD POST; 

        #为不同的响应状态码设置不同的缓存时间。
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404      1m;


        #设置响应被缓存的最小请求次数,最少2次才会缓存
        proxy_cache_min_uses 1;

        #开启此功能时，对于相同的请求，同时只允许一个请求发往后端
        proxy_cache_lock on; 

        #为proxy_cache_lock指令设置锁的超时5s
        proxy_cache_lock_timeout 5s;

        #忽略服务器不缓存的要求
        proxy_ignore_headers Cache-Control; 
    }


    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    }

}
```

#### 测试效果


```
// server1.php
<?php
$http = new Swoole\Http\Server("127.0.0.1", 9501);
$http->on('request', function ($request, $response) {
    echo "no cache".PHP_EOL;
    $response->end("<h1>9501</h1>");
});
$http->start();
```

> php server1.php #查看控制台输出

发送get和post请求

> ab -n10 -c10 http://nginx-t.com/v\=get

> ab -p 'data.json' -n10 -c10 http://nginx-t.com/v\=post

重复提交几次put请求

> curl -X PUT http://nginx-t.com/v\=put 

下面是我的测试结果截图（为了方便查看，我在get和post请求之前敲了几个空行）

![Nginx 缓存](https://cdn.learnku.com/uploads/images/201911/20/41489/StGI2M5SyZ.png!/fw/1240)

下面说明几个参数：

#### proxy_cache_path

```
语法:	proxy_cache_path path [levels=levels] keys_zone=name:size [inactive=time] [max_size=size] [loader_files=number] [loader_sleep=time] [loader_threshold=time];
默认值:	—
上下文:	http
```

- path：缓存数据是保存在文件中的，缓存的键和文件名都是在代理URL上执行MD5的结果。
- levels：定义了缓存的层次结构 
```
#当levels=1:2时，表示是两级目录，1和2表示用1位和2位16进制来命名目录名称。在此例中，第一级目录用1位16进制命名，如b；第二级目录用2位16进制命名，如2c。所以此例中一级目录有16个，二级目录有16*16=256个：
cache/b/2c/c75ad5e343f042f52e875343425e51b
```
- key_zone:在共享内存中设置一块存储区域来存放缓存的key和metadata(类似使用次数)，这样nginx可以快速判断一个request是否命中或者未命中缓 存，1m可以存储8000个key，10m可以存储80000个key。
- max_size:最大cache空间，如果不指定，会使用掉所有disk space，如果超过max_size参数设置的最大值，使用LRU算法移除缓存数据
- inactive:未被访问文件在缓存中保留时间，默认是10分钟。指定时间内未被访问的缓存文件将被删除。
- loader_files:每次最多加载的数量
- loader_sleeps:每次加载的延时
- loader_threshold:指定每次加载执行的时间

#### proxy_cache_lock

开启此功能时，对于相同的请求，同时只允许一个请求发往后端，并根据proxy_cache_key指令的设置在缓存中植入一个新条目。其他请求相同条目的请求将一直等待，直到缓存中出现相应的内容，或者锁在proxy_cache_lock_timeout指令设置的超时后被释放。

#### proxy_cache_valid

如果仅仅指定了time，
> proxy_cache_valid 5m;

那么只有状态码为200、300和302的响应会被缓存。

如果使用了any参数，那么就可以缓存任何响应：

> proxy_cache_valid any 1m;


#### proxy_ignore_headers
```
语法:	proxy_ignore_headers field ...;
默认值:	—
上下文:	http, server, location
```

不处理后端服务器返回的指定响应头。下面的响应头可以被设置： “X-Accel-Redirect”，“X-Accel-Expires”，“X-Accel-Limit-Rate” ，“X-Accel-Buffering” ， “X-Accel-Charset”，“Expires”，“Cache-Control”，和“Set-Cookie” 。

> 此参数不建议设置，原则上这些缓存应当后端代码处理。

#### proxy_cache_use_stale
```
语法:	proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_502 | http_503 | http_504 | http_404 | off ...;
默认值:	
proxy_cache_use_stale off;
上下文:	http, server, location
```

如果后端服务器出现状况，nginx是可以使用过期的响应缓存的。这条指令就是定义何种条件下允许开启此机制。这条指令的参数与proxy_next_upstream指令的参数相同。

#### proxy_cache_bypass与proxy_no_cache

```
语法:	proxy_cache_bypass string ...;
默认值:	—
上下文:	http, server, location
```
定义nginx不从缓存取响应的条件。如果至少一个字符串条件非空而且非“0”，nginx就不会从缓存中去取响应：
```
proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;
proxy_cache_bypass $http_pragma    $http_authorization;
```

本指令可和与proxy_no_cache一起使用。
```
语法:	proxy_no_cache string ...;
默认值:	—
上下文:	http, server, location
```
定义nginx不将响应写入缓存的条件。如果至少一个字符串条件非空而且非“0”，nginx就不将响应存入缓存：
```
proxy_no_cache $cookie_nocache $arg_nocache$arg_comment;
proxy_no_cache $http_pragma    $http_authorization;
```


#### proxy_cache_methods

该指令用于设置缓存哪些HTTP方法,默认缓存HTTP GET/HEAD方法,不缓存HTTP POST 方法。

有了代理缓存，那么清除缓存如何操作呢？

## 清除缓存

1. 删除缓存目录的文件
2. 使用ngx_cache_purge模块，[可查看这篇文章Nginx缓存配置及nginx ngx_cache_purge模块的使用](https://www.cnblogs.com/Eivll0m/p/4921829.html)

推荐第二种方法。

