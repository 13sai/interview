Nginx内嵌变量是非常常用的，记录下备查。

Nginx内嵌变量由 `ngx_http_core_module` 模块支持，变量名与Apache服务器对应，这些变量可以表示客户端的请求头字段，诸如`$http_user_agent`、`$http_cookie`等等。 nginx也支持其他变量：

参数名称 |说明
---|---
$arg_name|请求中的的参数名，即“?”后面的arg_name=arg_value形式的arg_name，如/index.php?www=www.13sai.com，可以用$arg_www就是www.13sai.com
$args|请求中的参数值
$binary_remote_addr|客户端地址的二进制形式, 固定长度为4个字节
$body_bytes_sent|传输给客户端的字节数，响应头不计算在内；这个变量和Apache的mod_log_config模块中的“%B”参数保持兼容
$bytes_sent|传输给客户端的字节数 
$connection|TCP连接的序列号
$connection_requests|TCP连接当前的请求数量 
$content_length|“Content-Length” 请求头字段
$content_type|“Content-Type” 请求头字段
$cookie_name|cookie名称
$document_root|当前请求的文档根目录或别名
$document_uri|同 $uri
$host|优先级如下：HTTP请求行的主机名>”HOST”请求头字段>符合请求的服务器名
$hostname|主机名
$http_name|匹配任意请求头字段； 变量名中的后半部分“name”可以替换成任意请求头字段，如在配置文件中需要获取http请求头：“Accept-Language”，那么将“－”替换为下划线，大写字母替换为小写，形如：$http_accept_language即可。
$https|如果开启了SSL安全模式，值为“on”，否则为空字符串。
$is_args|如果请求中有参数，值为“?”，否则为空字符串。
$limit_rate|用于设置响应的速度限制，详见 limit_rate。
$msec|当前的Unix时间戳 (1.3.9, 1.2.6)
$nginx_version|nginx版本
$pid|工作进程的PID
$pipe|如果请求来自管道通信，值为“p”，否则为“.” (1.3.12, 1.2.7)
$proxy_protocol_addr|获取代理访问服务器的客户端地址，如果是直接访问，该值为空字符串。(1.5.12)
$query_string|同 $args，然而 $query_string是只读的不会改变
$realpath_root|当前请求的文档根目录或别名的真实路径，会将所有符号连接转换为真实路径。
$remote_addr|客户端地址
$remote_port|客户端端口
$remote_user|用于HTTP基础认证服务的用户名
$request|代表客户端的请求地址
$request_body|客户端的请求主体,此变量可在location中使用，将请求主体通过proxy_pass, fastcgi_pass, uwsgi_pass, 和 scgi_pass传递给下一级的代理服务器。
$request_body_file|请求正文的临时文件名。处理完成时，临时文件将被删除。 如果希望总是将请求正文写入文件，需要开启client_body_in_file_only。 如果在被代理的请求或FastCGI请求中传递临时文件名，就应该禁止传递请求正文本身。 使用proxy_pass_request_body off指令 和fastcgi_pass_request_body off指令 分别禁止在代理和FastCGI中传递请求正文。
$request_completion|如果请求成功，值为”OK”，如果请求未完成或者请求不是一个范围请求的最后一部分，则为空。
$request_filename|当前连接请求的文件路径，由root或alias指令与URI请求生成。
$request_length|请求的长度 (包括请求的地址, http请求头和请求主体) 
$request_method|HTTP请求方法，通常为“GET”或“POST”
$request_time|处理客户端请求使用的时间; 从读取客户端的第一个字节开始计时。
$request_uri|这个变量等于包含一些客户端请求参数的原始URI，它无法修改，请查看$uri更改或重写URI，不包含主机名，例如：”/sai/test.php?arg=www”。
$scheme|请求使用的Web协议, “http” 或 “https”
$sent_http_name|可以设置任意http响应头字段； 变量名中的后半部分“name”可以替换成任意响应头字段，如需要设置响应头Content-length，那么将“－”替换为下划线，大写字母替换为小写，形如：$sent_http_content_length 4096即可。
$server_addr|服务器端地址，需要注意的是：为了避免访问linux系统内核，应将ip地址提前设置在配置文件中。
$server_name|服务器名
$server_port|服务器端口
$server_protocol|服务器的HTTP版本, 通常为 “HTTP/1.0” 或 “HTTP/1.1”
$status|HTTP响应代码
$time_iso8601|服务器时间的ISO 8610格式 
$time_local|服务器时间（LOG Format 格式） ，nginx处理完成打印日志的时间，不是请求发出的时间
$uri|请求中的当前URI(不带请求参数，参数位于$args)，可以不同于浏览器传递的$request_uri的值，它可以通过内部重定向，或者使用index指令进行修改，$uri不包含主机名，如”/foo/bar.html”。


apache服务器变量可看[[备忘] apache服务端变量](https://blog.csdn.net/litwhy/article/details/70893084)

参考：
- [中文文档](http://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_core_module.html#variables)

