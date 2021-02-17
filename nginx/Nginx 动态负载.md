如果Nginx遇到大流量和高负载，修改配置文件重启可能并不总是那么方便，因为恢复Nginx并重载配置会进一步增加系统负载，并很可能暂时降低性能。而一个个修改配置文件也是很容易出错和费时间的操作。


这时候不妨试试consul+nginx-upsync-module实现Nginx的动态负载。


## nginx-upsync-module

nginx-upsync-module 提供了动态的负载均衡，它可以从consul或etcd同步upstreams，动态修改后端服务器属性（weight，max_fails，down…），而不需要重新加载nginx。这样我们通过它实现平滑伸缩，而不严重地影响性能。

#### 利用docker安装

我已经基于centos7构建了一个镜像 13sai/nginx-lua-upsync ，你可以使用下面的命令启动一个容器

> docker run -itd --name=nginx-upsync -p 8008:80 -p 9501:9501 -p 9502:9502 -p 9503:9503 -p 8500:8500 13sai/nginx-lua-upsync


当然，你也可以不使用docker自行搭建，添加nginx-upsync-module模块可以参考[nginx模块lua模块](https://learnku.com/articles/36567)。

- [nginx-upsync-module的git地址](https://github.com/weibocom/nginx-upsync-module#installation)


#### 进入容器配置

> docker exec -it nginx-upsync /bin/bash

> cd /usr/local/nginx/conf

> echo "server host.docker.internal:9501 weight=1 fail_timeout=10 max_fails=3;" >> servers.conf

 

 > vi nginx.conf

 ```
 #nginx.conf 主要配置
 ...
 
 upstream 13sai{
    upsync 192.168.65.2:8500/v1/kv/upstreams/test-server upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
    upsync_dump_path /usr/local/nginx/conf/servers.conf;
    include /usr/local/nginx/conf/servers.conf;
}


server {
    listen       80;

    location / {
		proxy_pass http://13sai;
    }


    ...
    
}

...

 ```

#### upsync语法说明

```
语法：syntax: upsync $consul/etcd.api.com:$port/v1/kv/upstreams/$upstream_name/ [upsync_type=consul/etcd] [upsync_interval=second/minutes] [upsync_timeout=second/minutes] [strong_dependency=off/on]
默认值：无，如果省略参数，则默认参数为upsync_interval = 5s upsync_timeout = 6m strong_dependency = off
描述：从 consul/etcd 中拉取upstreams

upsync 定义从consul/etcd拉取最新的upstream信息并存到本地的操作
upsync_timeout 定义从consul/etcd拉取配置的超时时间
upsync_interval 定义从consul/etc拉取配置的间隔时间
upsync_type 定义使用配置服务类型
strong_dependency 启动时是否强制依赖配置服务器，如果配置为on,则拉取失败，nginx同样会启用失败
upsync_dump_path 定义从consul/etcd拉取配置后持久化到的本地的文件路径，这样即使 consul/etcd出问题了，本地同样会有备份文件
```

> 注意下面这个文件必须要有，文件路径和名称可以自定义，nginx-upsync-module会将负载信息缓存到此文件，否则Nginx启动会报错。

```
#servers.conf，192.168.x.xxx是我的宿主机ip
server 192.168.x.xxx:9501 weight=20 max_fails=1 fail_timeout=5s;
```


重启nginx

> /usr/local/nginx/sbin/nginx -t

> /usr/local/nginx/sbin/nginx -s reload

这里虽然我们还未启动consul，但没有什么影响，upsync会去拉取，也必然会失败，servers.conf就不会更新，Nginx的error日志会有信息。

## 利用swoole启动3个http服务

```
// 可启动3个server，端口分别为9501，9502，9503，输出也做对应修改
$http = new Swoole\Http\Server("127.0.0.1", 9501);
$http->on('request', function ($request, $response) {
    $response->end("9501");
});
```

## consul安装

> 这里consul只做一个kv存储，我自己也是第一次用，就不去做过多介绍了。

[下载地址](https://www.consul.io/downloads.html)

解压到你需要的目录，主要也就是一个consul可执行文件。（这里我装在我的电脑，而不是刚才的docker容器）

命令可看文档：[Consul 简介和快速入门](https://book-consul-guide.vnzmi.com/)


#### 启动：

> nohup ./consul agent -dev &

为了方便，我们也没有启动集群，生产环境建议使用consul集群。

#### UI查看

> http://127.0.0.1:8500/

#### 查看节点

> ./consul members

> curl 127.0.0.1:8500/v1/catalog/nodes


#### 查看kv值

> curl -v http://127.0.0.1:8500/v1/kv/\?recurse


![consul](https://cdn.learnku.com/uploads/images/201912/01/41489/aHdCOBhIWB.png!large)

#### 添加

> curl -X PUT -d '{"weight":20,"max_fails":2,"fail_timeout":5}' http://127.0.0.1:8500/v1/kv/upstreams/test-server/192.168.x.xxx:9502

此处192.168.x.xxx是因为我创建的docker容器的宿主机ip。

#### 删除

> curl -X DELETE http://127.0.0.1:8500/v1/kv/upstreams/test-server/192.168.x.xxx:9502

---

我们可以通过添加和删除来测试，查看http://127.0.0.1:8008/来查看输出，也可以看看Nginx里的配置文件servers.conf，你会看到你操作consul，会动态改变Nginx的upstream，这样就实现了Nginx的动态扩容。



