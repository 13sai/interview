## 安装lua

```
wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz 
tar -zxvf LuaJIT-2.0.5.tar.gz
cd LuaJIT-2.0.5
make && make install PREFIX=/usr/local/LuaJIT
```

## etc/profile加入
```
# lua
export LUAJIT_LIB=/usr/local/LuaJIT/lib 
export LUAJIT_INC=/usr/local/LuaJIT/include/luajit-2.0
```
> source etc/profile

## 下载ngx_devel_kit模块
```
wget https://github.com/simpl/ngx_devel_kit/archive/v0.3.0.tar.gz
```

> NDK(nginx development kit)模块是一个拓展nginx服务器核心功能的模块，第三方模块开发可以基于它来快速实现。 NDK提供函数和宏处理一些基本任务， 减轻第三方模块开发的代码量


## 下载lua-nginx-module模块
```
 wget https://github.com/openresty/lua-nginx-module/archive/v0.10.9rc7.tar.gz 
```
 lua-nginx-module 模块使nginx中能直接运行lua

## 查看原始编译

> nginx -V


```
如：
configure arguments: --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_gzip_static_module --with-http_sub_module --with-http_v2_module
```


进入nginx原始目录：


```
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_gzip_static_module --with-http_sub_module --with-http_v2_module --add-module=/root/lua-nginx-module-0.10.9rc7/ --add-module=/root/ngx_devel_kit-0.3.0
```

只make，不执行make install。

编译报错应该就是lua环境变量不对。


```
nginx -V 命令报错
./nginx: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory

解决：
echo "/usr/local/LuaJIT/lib" >> /etc/ld.so.conf

ldconfig

```

成功之后可以nginx -V查看，无报错即可。

把原来的nginx备份为nginx_old

cp objs/nginx到原来的nginx并覆盖。

在编译目录执行
> make upgrade

![Nginx 添加 lua 模块](https://cdn.learnku.com/uploads/images/201911/18/41489/SNDXytBYrH.png!large)


测试：
```
server{
    ...
    location /lua {
        default_type 'text/html';
        content_by_lua '
                ngx.say("hello, lua!")
        ';
    }
    ...
}
```


浏览器打开：
> http://blog.13sai.com/lua


可以看到hello, lua!

