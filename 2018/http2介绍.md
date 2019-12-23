标签：http2 iplas

---

为什么要使用HTTP2
===========

原因就是慢
---

     影响一个网络请求的因素主要有两个，带宽和延迟。今天的网络基础建设已经使得带宽得到极大的提升，大部分时候都是延迟在影响。
     
     **为什么延迟？？**

 - 浏览器阻塞（HOL blocking）：浏览器会因为一些原因阻塞请求。浏览器对于同一个域名，同时只能有 4
   个连接（这个根据浏览器内核不同可能会有所差异），超过浏览器最大连接数限制，后续请求就会被阻塞。
 - DNS 查询（DNS Lookup）：浏览器需要知道目标服务器的 IP 才能建立连接。将域名解析为 IP的这个系统就是 DNS。这个通常可以利用DNS缓存结果来达到减少这个时间的目的。
 - 建立连接（Initial connection）：HTTP 是基于 TCP 协议的，浏览器最快也要在第三次握手时才能捎带 HTTP 请求报文，达到真正的建立连接，但是这些连接无法复用会导致每次请求都经历三次握手和慢启动。三次握手在高延迟的场景下影响较明显，慢启动则对文件类大请求影响较大。
 - 连接无法复用。每回请求个资源就会有一个请求连接跟断开连接操作，大量请求不断的请求连接跟断开连接时间开销大。

如何高响应速度？pipelining的出现
------

![HTTP1.0 -> HTTP1.1][1]
前文介绍，HTTP 连接无法复用会导致每次请求都经历三次握手和慢启动。由图可知pipelining可有效减少握手时间。

不过pipelining并不是救世主，它也存在不少缺陷：

 1. pipelining只能适用于http1.1，一般来说，支持http1.1的server都要求支持pipelining
 2. 只有幂等的请求（GET，HEAD）能使用pipelining，非幂等请求比如POST不能使用，因为请求之间可能会存在先后依赖关系。
 3. head of line blocking并没有完全得到解决，server的response还是要求依次返回，遵循FIFO(first in first out)原则。也就是说如果请求1的response没有回来，2，3，4，5的response也不会被送回来。
 4. 绝大部分的http代理服务器不支持pipelining。
 5. 和不支持pipelining的老服务器协商有问题。
 6. 可能会导致新的Front of queue blocking问题。

    

普通的 HTTPS 网站浏览会比 HTTP 网站稍微慢一些，因为需要处理加密任务。
-----------------------------------------

HTTP2 VS HTTP1.1
================

 1. 多路复用。
    多路复用通过多个请求stream共享一个tcp连接的方式（即所有的HTTP2.0的请求都在一个TCP链接上），解决了http1.x holb（head of line blocking）的问题，降低了延迟同时提高了带宽的利用率。
 ![此处输入图片的描述][2]
 2. 压缩头部。
    HTTP/2.0规定了在客户端和服务器端会使用并且维护「首部表」来跟踪和存储之前发送的键值对，对于相同的头部，不必再通过请求发送，只需发送一次。
如果首部发生变化了，那么只需要发送变化了数据在Headers帧里面，新增或修改的首部帧会被追加到“首部表”。首部表在 HTTP2.0的连接存续期内始终存在,由客户端和服务器共同渐进地更新。 
由于减少了大量请求头数据的传输，缓解了网络压力，也提高了响应速度。
![此处输入图片的描述][3]
 3. 二进制分帧
 在应用层与传输层之间增加一个二进制分帧层，以此达到“在不改动HTTP的语义，HTTP 方法、状态码、URI及首部字段的情况下，突破HTTP1.1的性能限制，改进传输性能，实现低延迟和高吞吐量。”
在二进制分帧层上，HTTP2.0会将所有传输的信息分割为更小的消息和帧,并对它们采用二进制格式的编码，其中HTTP1.x的首部信息会被封装到Headers帧，而我们的request body则封装到Data帧里面。
![此处输入图片的描述][4]
 4. 并行双向字节流的请求和响应
    在HTTP2.0上，客户端和服务器可以把HTTP消息分解为互不依赖的帧，然后乱序发送，最后再在另一端把它们重新组合起来。注意，同一链接上有多个不同方向的数据流在传输。客户端可以一边乱序发送stream，也可以一边接收服务器的响应，而服务器那端同理。
![此处输入图片的描述][5]
 5. 请求优先级
多路复用导致所有资源都是并行发送，那么就需要「优先级」的概念了，这样就可以对重要的文件进行先传输，加速页面的渲染。
 6. 服务器推送
服务器推送是指在客户端请求之前发送数据的机制。（根据请求资源猜测你可能还需要某某资源，提前推送资源缓存提高响应效率）

 

nginx升级HTTP2
============

###1、升级准备 
```
1、安装编译器
yum -y install gc gcc gcc-c++   pcre* perl*
2、下载安装包
cd /usr/local
wget http://nginx.org/download/nginx-1.12.2.tar.gz
wget https://www.openssl.org/source/openssl-1.0.2e.tar.gz
wget http://zlib.net/zlib-1.2.11.tar.gz
3、解压安装包
tar -zxvf nginx-1.12.2.tar.gz
tar -zxvf openssl-1.0.2e.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
```
###2、安装zlib
>*1、cd zlib
>  2、./configure
>  3、make
>  4、make install
###3、安装openssl
>1、cd openssl
>##2、安装
>    ./config shared zlib
>    make
>    make install
>    cd /usr/local/ssl/
>    ./bin/openssl version -a
>##3、替换旧版OpenSSL
>    mv /usr/bin/openssl /usr/bin/openssl.old
>    mv /usr/include/openssl /usr/include/openssl.old
>    ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
>    ln -s /usr/local/ssl/include/openssl/ /usr/include/openssl
>##4、配置库文件搜索路径
>    echo "/usr/local/ssl/lib" >> /etc/ld.so.conf
>    ldconfig
>##5、测试新版本的OpenSSL是否正常工作
>    openssl version -a
###4、安装nginx
>##1、查看ngixn版本及其编译参数，复制你已经编译安装好的模块。
>    nginx -V
>##2、进入nginx源码目录
>    cd nginx
>    执行（在原有的参数上加上--with-zlib=/usr/local/zlib-1.2.11    --with-openssl=/usr/local/openssl-1.0.2e重编译，如下例：）
>    ./configure  --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module   --with-zlib=/usr/local/zlib-1.2.11 --with-openssl=/usr/local/openssl-1.0.2e  --with-pcre
>##3、然后
>    make   //千万别make install，否则就覆盖安装了
>    make完之后在objs目录下就多了个nginx，这个就是新版本的程序了
>##4、验证新nginx是否可用验证编译后的nginx是否可以使用已有的配置
```
./objs/nginx -t
```
>##5、备份旧的nginx：
```
 cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```
>##6、把新的nginx程序覆盖旧的：
```
 cp ./objs/nginx /usr/sbin/nginx
```
>##7、测试新的nginx程序是否正确
```
nginx -t
```
>##8、修改Nginx的 .conf 文件
>    在监听443的后面加上http2，如下：
>    ```
>    server {
>    	listen 443 ssl http2 default_server;
>    	server_name www.jiezaizone.cn; 
>    	ssl on;
>    	ssl_certificate /etc/nginx/1_jiezaizone.cn_bundle.crt;
>    	ssl_certificate_key /etc/nginx/2_jiezaizone.cn.key;
>    	ssl_session_timeout 5m;
>    	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
>    	ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
>    	ssl_prefer_server_ciphers on;
>    	location / {
>    		root   /; 
>    		index  index.html index.htm;
>    	}
>    }
>    ```
>    注：某些服务开启http2后不能访问，在http2 后面加上 default_server 就可以了
>##9、重新加载nginx
```
    nginx -s reload
```
###安装异常及其解决方法
>异常1：
>/usr/bin/ld: /usr/local/lib/libz.a(crc32.o): relocation R_X86_64_32 against `a local symbol' can not be used when making a shared object; recompile with -fPIC

>/usr/local/lib/libz.a: could not read symbols: Bad value
>处理方法:
>cd zlib-1.2.3 //进入zlib目录

>CFLAGS="-O3 -fPIC" ./configure   //使用64位元的方法进行编译

>make

>异常2：
>/usr/bin/ld: /etc/nginx/openssl-1.0.2e/.openssl/lib/libssl.a(s23_meth.o): relocation R_X86_64_32 against `.rodata' can not be used when making a shared object; recompile with -fPIC
>    解决方法：
>    cd openssl
>    ./config -fPIC --prefix=/usr/local/openssl/ enable-shared
>    make

​    

测试环境运营系统已经全面支持https
===================
1、仓库已经push了nginx1.11.1，直接支持http2，可通过下述命令直接拉取下拉使用
docker pull docker.oa.isuwang.com:5000/system/nginx:1.11.1
2、在我们的nginx项目http2分支，已经写好了构建支持http2的nginx dockerdile，可直接下载下拉查看。
http://git.oa.isuwang.com/isuwang-docker/nginx.git

提问：
1. HTTPS有并发吗，如何实现的？一个连接怎么实现并发呢
2. http2的长连接是怎么回事

答：
1. http2有并发。它支持多路复用，并行双向字节流的请求和响应，它以“流”的形式在客户端和服务器间独立的双向的交换的帧序列。流具有一些重要的特性：
 - 单个的HTTP/2连接可以包含多个并发打开的流，各个终端多个流的帧可以交叉。
 - 流可以单方面地建立和使用，或由客户端或服务器共享。
 - 流可以被任何一端关闭。
 - 流中帧的发送顺序是值得注意的。接收者以它们收到帧的顺序处理。特别的，HEADERS帧和DATA帧在语义上是非常重要的。
 - 流由一个整数标识。流标识符由发起流的一端来赋值。

虽然只有一个TCP连接，却能够在这个连接同时处理客户端和服务端的多个数据传输。
2. HTTP2只有一个TCP连接，它并不像http1.1中的TCP连接一样，在处理当前传输数据后就关闭，而是一直连通的，客户端和服务端可以通过它持续往返传输数据。这个TCP流的传输通道，是可以被任何一端给关闭的。

@By：刘荣杰


[1]: https://sfault-image.b0.upaiyun.com/730/391/730391669-573002ce11248_articlex
[2]: https://sfault-image.b0.upaiyun.com/126/067/1260679140-573002cec3232_articlex
[3]: http://7xs2h9.com1.z0.glb.clouddn.com/blog/Header%E5%A4%8D%E7%94%A8.png
[4]: http://7xs2h9.com1.z0.glb.clouddn.com/blog/%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%88%86%E5%B8%A7.png
[5]: http://i.imgur.com/WDzDClq.png