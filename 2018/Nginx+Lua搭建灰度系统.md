---
title: Nginx+Lua搭建灰度系统
date: 2018-07-09 14:09:15
tags:
categories: Nginx+Lua研究与学习
---
###一、环境准备

##### 1、直接docker下载打包好的镜像 (这里默认大家都已经安装了docker)
```text
## https://hub.docker.com/r/openresty/openresty/ 这里可以了解一下dockerfile
$ docker search openresty
$ docker pull openresty/openresty 
```
##### 2、启动容器(这里用9000代替80端口，避免冲突，9000端口可以随意改)
```text
$ docker run -it --name openresty -d -p9000:80 openresty/openresty
## 容器外输入（localhost换成对应的ip 或者直接浏览器访问 ）
$ curl localhost:9000
## 会看到 Welcome to OpenResty!
## 那就是成功了 ：）
```
##### 3、现在我们进入容器进行灰度系统搭建
```text
## 命令行输入
$ docker exec -it openresty sh
$ vi /usr/local/openresty/nginx/conf/nginx.conf
```
```text
## vi内操作
##在include mime.types;下面增加以下配置
##lua模块路径，其中”;;”表示默认搜索路径
lua_package_path "/usr/local/openresty/lualib/?.lua;;"; #lua模块                  
lua_package_cpath "/usr/local/openresty/lualib/?.so;;"; #c模块                  
include /usr/local/app/conf/lua.conf; #引用自定义的配置文件
## 最后把那堆server删掉
## wq 保存退出
```
```text
## 命令行输入
$ mkdir -p /usr/local/app/conf/
$ cd /usr/local/app/conf/
$ vi lua.conf
```
```text
## vi内添加
server{

  listen 80;
  
  location /hello_world{
      default_type 'text/html';
      content_by_lua 'ngx.say("hello world :)")'
  }
}
## wq
```
```text
$ /usr/local/openresty/bin/openresty -c /usr/local/openresty/nginx/conf/nginx.conf -s reload
## 为了简便 我们进行如下操作：
$ mkdir /usr/local/app/bin/
$ vi /usr/local/app/bin/reload.sh
## 内容大概是这样的
#!/bin/bash
/usr/local/openresty/bin/openresty -c /usr/local/openresty/nginx/conf/nginx.conf -s reload
```
```text
##容器外 或 浏览器输入
$ curl http://localhost:9000/hello_world
### hello world :)
```
##### 恭喜你基础的环境已经搭建好了！

### 二、连上的redis
##### 1、在lua.conf 中增加配置如下：
```text
location /demo1{
  default_type 'text/html';
  content_by_lua_file /usr/local/app/lua/demo1.lua;
}
```
```text
$ mkdir /usr/local/app/lua/
$ vi /usr/local/app/lua/demo1.lua
```
```java
local function close_redis(red)  
    if not red then  
        return  
    end  
    --释放连接(连接池实现)  
    local pool_max_idle_time = 10000 --毫秒  
    local pool_size = 100 --连接池大小  
    local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)  
    if not ok then  
        ngx.say("set keepalive error : ", err)  
    end  
end  
  
local redis = require("resty.redis")  
  
-- 创建实例  
local red = redis:new()  
-- 设置超时（毫秒）  
red:set_timeout(1000)  
-- 建立连接  
local ip = "192.168.4.22"  
local port = 6379  
local ok, err = red:connect(ip, port)  
if not ok then  
    ngx.say("connect to redis error : ", err)  
    return close_redis(red)  
end  
-- 调用API进行处理  
ok, err = red:set("msg", "hello world")  
if not ok then  
    ngx.say("set msg error : ", err)  
    return close_redis(red)  
end  
  
-- 调用API获取数据  
local resp, err = red:get("msg")  
if not resp then  
    ngx.say("get msg error : ", err)  
    return close_redis(red)  
end  
-- 得到的数据为空处理  
if resp == ngx.null then  
    resp = ''  --比如默认值  
end  
ngx.say("msg : ", resp)  
  
close_redis(red)
```
#### 业务场景：通过配置改变某个用户的代理ip，从而达到不同用户发送到不同的服务的目的
##### 现在看看demo2
```java
local res,err = ngx.location.capture("/redis/hget",{args={key="gray",mKey="16218"}})
local parser = require "redis.parser"
local server,type = parser.parse_reply(res.body)
if not server then
    server="default"
end
ngx.say("server is :" .. server)
```
##### demo3
```java
local res,err = ngx.location.capture("/redis/hget",{args={key="gray",mKey="16218"}})
local parser = require "redis.parser"
local server,type = parser.parse_reply(res.body)
if not server then
    server="default"
end
ngx.target = server
return
```
```text
## /usr/local/openresty/nginx/conf/nginx.conf
## 在http下添加
upstream redis_server{
  server 192.168.4.22:6379;
  keepalive 1024;
}
##并且在woker_processes 下面增加日志
error_log /usr/local/logs/error.log
```
```text
## /usr/local/openresty/nginx/conf/nginx.conf
  location /demo2{
    default_type 'text/html';
    content_by_lua_file /usr/local/app/lua/demo2.lua;
  }
  
  location /demo3{
    set $target '';
    access_by_lua_file /usr/local/app/lua/demo3.lua;
    proxy_pass http://$target;
  }
  
  location /redis/hget{
    internal; #只允许内部调用
    redis2_query hget $arg_key $arg_mKey;
    redis2_pass redis_server;
  }
```
###### demo2，demo3先看看官方文档怎么写：
```text
This module can be served as a non-blocking redis2 client for lua-nginx-module
(but nowadays it is recommended to use the lua-resty-redis library instead, 
which is much simpler to use and more efficient most of the time). 
##大概意思就是lua对redis的操作更推荐使用lua-resty-redis模块（因为更简单，更高效）
##但是这里只是为了方便演示，使用demo3的写法
```
###### github地址
```text
git clone https://github.com/JMMMM/nginxLua.git
```
### 参考文献
Openresty最佳实践 https://moonbingbing.gitbooks.io/openresty-best-practices/content/ </Br>
跟我学OpenResty(Nginx+Lua)http://jinnianshilongnian.iteye.com/blog/2190344 </br>
官方文档 http://openresty.org/cn/

@伍嘉明