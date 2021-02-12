PHP-FPM(PHP FastCGI Process Manager)：PHP FastCGI 进程管理器，用于管理PHP 进程池的软件，用于接受web服务器的请求。



主要配置说明：

```
[www]
user = www               #进程的发起用户和用户组，用户user是必须设置，group不是,nobody 任意用户
group = www

listen = [::]:9000          #监听ip和端口，[::] 代表任意ip
chdir = /app                #在程序启动时将会改变到指定的位置(这个是相对路径，相对当前路径或chroot后的“/”目录)　

pm = dynamic                #选择进程池管理器如何控制子进程的数量  static：　　对于子进程的开启数路给定一个锁定的值(pm.max_children)   dynamic:　 子进程的数目为动态的，它的数目基于下面的指令的值(以下为dynamic适用参数)
pm.max_children = 16        #同一时刻能够存货的最大子进程的数量，static的制定数量
pm.start_servers = 4        #在启动时启动的子进程数量
pm.min_spare_servers = 2    #处于空闲"idle"状态的最小子进程，如果空闲进程数量小于这个值，那么相应的子进程会被创建
pm.max_spare_servers = 16   #最大空闲子进程数量，空闲子进程数量超过这个值，那么相应的子进程会被杀掉。

catch_workers_output = Yes  #将worker的标准输出和错误输出重定向到主要的错误日志记录中，如果没有设置，根据FastCGI的指定，将会被重定向到/dev/null上
```

## pm的三种模式区别

1. pm=static：始终保持固定数量的worker进程数，由pm.max_children决定，不会动态扩容。

配置项要求：pm.max_children> 0 必须配置，且只有这一个参数生效

优缺点：如果配置成static，只需要考虑max_children的数量，数量取决于cpu的个数和应用的响应时间

2. pm=dynamic：php-fpm启动时，会初始启动一些worker，初始启动worker数决定于pm.max_children的值。在运行过程中动态调整worker数量，worker的数量受限于pm.max_children配置，同时受限全局配置process.max。

1秒定时器作用，检查空闲worker数量，按照一定策略动态调整worker数量，增加或减少。增加时，worker最大数量<=max_children· <=全局process.max；减少时，只有idle >pm.max_spare_servers时才会关闭一个空闲worker。

优点：动态扩容，不浪费系统资源
缺点：如果所有worker都在工作，新的请求到来只能等待master在1秒定时器内再新建一个worker，这时可能最长等待1s

3. pm=ondemand：php-fpm启动的时候，不会启动任何一个worker，而是按需启动，只有当连接过来的时候才会启动。

启动的最大worker数决定于pm.max_children的值，同时受限全局配置process.max。1秒定时器作用，如果空闲worker时间超过pm.process_idle_timeout的值（默认值为10s），则关闭该worker。这个机制可能会关闭所有的worker。

优点：按流量需求创建，不浪费系统资源
缺点：由于php-fpm是短连接的，所以每次请求都会先建立连接，频繁的创建worker会浪费系统开销。，所以，在大流量的系统上，master进程会变得繁忙，占用系统cpu资源，不适合大流量环境的部署。

## worker进程、master进程详解

master只是负责监听管理工作，并不是很多人认为的把客户端发来的请求分给worker进程处理，而是由**worker进程负责客户端的请求监听和处理**。这和nginx是不一样的，需要注意一下。

一旦kill掉worker进程后，会重启一个新的worker进程。因此客户端请求肯定会得到响应处理。master进程负责监听子进程的状态，子进程挂掉之后，会发信号给master进程，然后master进程重新启一个新的worker进程。