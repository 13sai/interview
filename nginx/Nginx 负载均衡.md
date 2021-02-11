## 负载均衡策略

- 1.基于轮询的均衡策略(默认)：
每一个来自网络中的请求，轮流分配给内部的服务器，从1到N然后重新开始。此种负载均衡算法适合服务器组内部的服务器都具有相同的配置并且平均服务请求 相对均衡的情况。

- 2.基于加权轮询的均衡策略（weight）：根据服务器的不同处理能力，给每个服务器分配不同的权值，使其能够接受相应权值数的服务请求。例如:服务器A的权值被设计成1，B的权值是3，C的权值是
6，则服务器A、B、C将分别接受到10%、30%、60%的服务请求。此种均衡算法能确保高性能的服务器得到更多的使用率，避免低性能的服务器负载过重。


- 3.基于ip-hash的均衡策略（ip_hash）：
我们都知道，每个请求的客户端都有相应的ip地址，该均衡策略中，nginx将会根据相应的hash函数，对每个请求的ip作为关键字，得到的hash值将会决定将请求分发给相应Server进行处理


- 4.基于最少连接数的均衡策略（least_conn）：
最少连接，也就是说nginx会判断后端集群服务器中哪个Server当前的 Active Connection 数是最少的，那么对于每个新进来的request,nginx将该request分发给对应的Server.


## 语法

```
语法: upstream name { ... } 
默认值: —
上下文: http


upstream 指令当中包含server指令
语法: server address [parameters]; 
默认值: —
上下文: upstream


例子:
upstream backend {
    server backend1.example.com:8081 weight=4 max_fails=2 fail_timeout=30s; 
    server backend2.example.com:8080 weight=1;
}
server { 
    location / {
        proxy_pass http://backend; 
    }
}


参数说明:
weight=number 设定服务器的权重，默认是1，权重越大被访问机会越大，要根据机器的配置情况来配置

max_fails=number 设定Nginx与服务器通信的尝试失败的次数。在fail_timeout参数定义的时间段内，如果失败的次数达到此值，Nginx就认为服务器不 可用。在下一个fail_timeout时间段，服务器不会再被尝试。 失败的尝试次数默认是1。

可以通过指令proxy_next_upstream 和memcached_next_upstream来配置什么是失败的尝试。 默认配置时，http_404状态不被认为是失败的尝试。 

fail_timeout=time
统计失败尝试次数的时间段。在这段时间中，服务器失败次数达到指定的尝试次数，服务器就被认为不可用。默认情况下，该超时时间是10秒。 

backup
标记为备用服务器。当主服务器不可用以后，请求会被传给这些服务器，配置这个指令可以实现故障转移。

down 标记服务器永久不可用，可以跟ip_hash指令一起使用。
```

即便未设置诸多参数，默认情况下，当其中某个服务挂掉之后，nginx还是会把请求分发到正常的服务中去。但一般建议我们手动去设置。

##  proxy_next_upstream 指令
在nginx的配置文件中， proxy_next_upstream 项定义了什么情况下进行重试
```
语法: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...; 
默认值: proxy_next_upstream error timeout;
上下文: http, server, location


其中:
error 表示和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现错误。 

timeout 表示和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现超时。 

invalid_header 表示后端服务器返回空响应或者非法响应头

http_500 表示后端服务器返回的响应状态码为500

non_idempotent 通常，如果请求已发送到上游服务器，则具有非等幂方法（POST、LOCK、PATCH）的请求不会传递到下一个服务器；启用此选项可显式允许重试此类请求；

off 表示停止将请求发送给下一台后端服务器
```

注意下non_idempotent参数。

#### 相关

```
proxy_next_upstream_tries number:设置重试次数，默认0表示不限制，注意此重试次数指的是所有请求次数(包括第一次和之后的重试次数之和)。 

proxy_next_upstream_timeout time: 设置重试最大超时时间，默认0表示不限制。
即在 proxy_next_upstream_timeout 时间内允许 proxy_next_upstream_tries 次重试。如果超过了其中一个设置，则 Nginx 也会结束重试并返回客户 端响应(可能是错误码)。

proxy_send_timeout 后端服务器数据回传时间(代理发送超时时间)

proxy_read_timeout  连接成功后，后端服务器响应时间(代理接收超时时间) 

proxy_connect_timeout nginx连接后端的超时时间，一般不超过75s
```