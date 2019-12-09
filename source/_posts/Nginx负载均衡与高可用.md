---
layout: title
title: Nginx负载均衡与高可用
date: 2018-04-07 09:26:36
categories: Nginx
tags: Nginx
top: true
---
Nginx负载均衡和高可用的原理与实践

<!--more-->

# 快速跳转

* 配置
* 正向代理


# 正向代理

Nginx不仅可以做反向代理，实现负载均衡。还能用作正向代理来进行上网等功能。

正向代理：如果把局域网外的Internet想象成一个巨大的资源库，则局域网中的客户端要访问Internet，则需要通过代理服务器来访问，这种代理服务就称为正向代理。

{% asset_img 1.png %}

# 反向代理

反向代理，其实客户端对代理是无感知的，因为客户端不需要任何配置就可以访问，我们只需要将请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，在返回给客户端，此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器IP地址。

{% asset_img 2.png %}

# 负载均衡

客户端发送多个请求到服务器，服务器处理请求，有一些可能要与数据库进行交互，服务器处理完毕后，再将结果返回给客户端。

这种架构模式对于早期的系统相对单一，并发请求相对较少的情况下是比较适合的，成本也低。但是随着信息数量的不断增长，访问量和数据量的飞速增长，以及系统业务的复杂度增加，这种架构会造成服务器相应客户端的请求日益缓慢，并发量特别大的时候，还容易造成服务器直接崩溃。很明显这是由于服务器性能的瓶颈造成的问题，那么如何解决这种情况呢？

我们首先想到的可能是升级服务器的配置，比如提高CPU执行频率，加大内存等提高机器的物理性能来解决此问题，但是我们知道摩尔定律的日益失效，硬件的性能提升已经不能满足日益提升的需求了。最明显的一个例子，天猫双十一当天，某个热销商品的瞬时访问量是极其庞大的，那么类似上面的系统架构，将机器都增加到现有的顶级物理配置，都是不能够满足需求的。那么怎么办呢？

上面的分析我们去掉了增加服务器物理配置来解决问题的办法，也就是说纵向解决问题的办法行不通了，那么横向增加服务器的数量呢？这时候集群的概念产生了，单个服务器解决不了，我们增加服务器的数量，然后将请求分发到各个服务器上，将原先请求集中到单个服务器上的情况改为将请求分发到多个服务器上，将负载分发到不同的服务器，也就是我们所说的负载均衡。

{% asset_img 3.png %}

# 动静分离

为了加快网站的解析速度，可以把动态页面和静态页面由不同的服务器来解析，加快解析速度。降低原来单个服务器的压力。

{% asset_img 4.png %}

# 配置

可以将nginx.conf配置文件分为三部分：

**第一部分：全局块**

从配置文件开始到events块之间的内容，主要会设置一些影响nginx 服务器整体运行的配置指令，主要包括配置运行Nginx服务器的用户（组）、允许生成的worker process数，进程PID存放路径、日志存放路径和类型以及配置文件的引入等。

比如第一行配置的：
```txt
worker_processes 1;
```
这是Nginx服务器并发处理服务的关键配置，worker_processes值越大，可以支持的并发处理量也越多，但是会受到硬件、软件等设备的制约。

**第二部分：events块**

```txt
events{
    worker_connections 1024;
}
```
events块涉及的指令主要影响Nginx服务器与用户的网络连接，常用的设置包括是否开启对多work process下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个word process可以同时支持的最大连接数等。

上述例子就表示每个work process支持的最大连接数为1024。

这部分的配置对Nginx的性能影响较大，在实际中应该灵活配置。

**第三部分：http块**

```conf
http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

      
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```
这算是Nginx服务器配置中最频繁的部分，代理、缓存和日志定义等绝大多数功能和第三方模块的配置都在这里。

需要注意的是：http块也可以包括http全局块、server块。

<span style="color:red">①http全局块</span>

http全局块配置的指令包括文件引入、MIME-TYPE定义、日志自定义、连接超时时间、单链接请求数上限等。

<span style="color:red">②server块</span>

这块和虚拟主机有密切关系，虚拟主机从用户角度看，和一台独立的硬件主机是完全一样的，该技术的产生是为了节省互联网服务器硬件成本。

每个http块可以包括多个server块，而每个server块就相当于一个虚拟主机。

而每个server块也分为全局server块，以及可以同时包含多个locaton块。

**1、全局server块**

最常见的配置是本虚拟机主机的监听配置和本虚拟主机的名称或IP配置。

**2、location块**

一个server块可以配置多个location块。

这块的主要作用是基于Nginx服务器接收到的请求字符串（例如server_name/uri-string），对虚拟主机名称（也可以是IP别名）之外的字符串（例如 前面的 /uri-string）进行匹配，对特定的请求进行处理。地址定向、数据缓存和应答控制等功能，还有许多第三方模块的配置也在这里进行。

# 配置实例-反向代理

**实例一**

实现效果：使用nginx反向代理，访问www\.123.com直接跳转到127.0.0.1:8080

```txt
server{
    listen 80;
    server_name www.123.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        index index.html index.htm index.jsp;
    }
}
```

**实例二**

实现效果：使用nginx反向代理，根据访问的路径跳转到不同端口的服务中。

nginx监听端口为9001
访问http://127.0.0.1:9001/edu/ 直接跳转到 127.0.0.1:8001
访问http://127.0.0.1:9001/vod/ 直接跳转到 127.0.0.1:8002

```txt
server{
    listen 9001;
    server_name localhost;

    location ~ /edu/ {
        proxy_pass http://localhost:8001;
    }

    location ~ /vod/ {
        proxy_pass http://localhost:8002;
    }
}
```

<span style="color:red">location指令说明</span>

该指令用于匹配URL。
语法如下：
```txt
location [ = | ~ | ~* | ^~ ] uri{

}
```
1、=：用于不含正则表达式的uri前，要求请求字符串与uri严格匹配，如果匹配成功，就停止继续向下搜索并立即处理该请求。
2、\~：用于表示uri包含正则表达式，并且区分大小写。
3、\~\*：用于表示uri包含正则表达式，并且不区分大小写。
4、\^\~：用于不含正则表达式的uri前，要求Nginx服务器找到标识uri和请求字符串匹配度最高的location后，立即使用此location处理请求，而不再使用location块中的正则uri和请求字符串做匹配。

<span style="color:red">注意：如果uri包含正则表达式，则必须要有\~或者\~\*标识。</span>

# 配置实例-负载均衡

实现效果：配置负载均衡

```txt
http{
    ...
    upstream myserver{
        ip_hash;
        server 115.28.52.63:8080 weight=1;
        server 115.28.52.63:8180 weight=1;
    }
    ...

    server{
        location / {
            ...
            proxy_pass http://myserver;
            proxy_connect_timeout 10;
        }
        ...
    }
}
```
随着互联网信息的爆炸性增长，负载均衡（load balance）已经不再是一个很陌生的话题，顾名思义，负载均衡即是将负载分摊到不同的服务单元，既保证服务的可用性，又保证响应足够快，给用户很好的体验。快速增长的访问量和数据流量催生了各式各样的负载均衡产品，很多专业的负载均衡硬件提供了很好的功能，但却价格不菲，这使得负载均衡软件大受欢迎，nginx就是其中的一个，在linux下有Nginx、LVS、Haproxy等等服务可以提供负载均衡服务，而且Nginx提供了几种分配方式（策略）：

<span style="color:red">1、轮询（默认）</span>

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

<span style="color:red">2、weight</span>

weight代表权重，默认为1，权重越高被分配的客户端越多指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 例如：
```txt
upstream server_pool{
    server 192.168.5.21 weight=10;
    server 192.168.5.22 weight=10;
}
```

<span style="color:red">3、ip_hash</span>

每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。 例如：
```txt
upstream server_pool {
    ip_hash ;
    server 192.168.5.21:80;
    server 192.168.5.22:80;
}
```

<span style="color:red">4、fair（第三方）</span>

按后端服务器的响应时间来分配请求，响应时间短的优先分配。

```txt
upstream server_pool {
    server 192.168.5.21:80;
    server 192.168.5.22:80;
    fair ;
}
```

# 配置实例-动静分离

Nginx动静分离简单来说就是把动态跟静态请求分开，不能理解成只是单纯的把动态页面和静态页面物理分离。严格意义上说应该是动态请求跟静态请求分开，可以理解成使用Nginx处理静态页面，Tomcat处理动态页面。动静分离从目前实现角度来讲大致分为两种，一种是纯粹把静态文件独立成单独的域名，放在独立的服务器上，也是目前主流推崇的方案；

另外一种方法就是动态跟静态文件混合在一起发布，通过 nginx 来分开。

通过location指定不同的后缀名实现不同的请求转发。通过expires 参数设置，可以使浏览器缓存过期时间，减少与服务器之前的请求和流量。具体Expires定义：是给一个资源设定一个过期时间，也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可，所以不会产生额外的流量。此种方法非常适合不经常变动的资源。（如果经常更新的文件，不建议使用Expires来缓存），我这里设置3d，表示在这3天之内访问这个URL，发送一个请求，比对服务器该文件最后更新时间没有变化，则不会从服务器抓取，返回状态码304，如果有修改，则直接从服务器重新下载，返回状态码200。

```txt
server{
    listen 9001;
    server_name 192.168.17.129;

    location /www/ {
        root /data/;
        index index.html index.htm;
    }

    location /image/ {
        root /data/;
        autoindex on;
    }
}
```

重点是添加location

最后检查Nginx配置是否正确即可，然后测试动静分离是否成功，之需要删除后端tomcat服务器上的某个静态文件，查看是否能访问，如果可以访问说明静态资源nginx直接返回了，不走后端tomcat服务器

# nginx原理与优化参数配置

{% asset_img 5.png %}

{% asset_img 6.png %}

**master-workers的机制的好处**

首先，对于每个worker进程来说，独立的进程，不需要加锁，所以省掉了锁带来的开销，同时在编程以及问题查找时，也会方便很多。其次，采用独立的进程，可以让互相之间不会影响，一个进程退出后，其它进程还在工作，服务不会中断，master进程则很快启动新的worker进程。当然，worker 进程的异常退出，肯定是程序有 bug 了，异常退出，会导致当前worker上的所有请求失败，不过不会影响到所有请求，所以降低了风险。

**需要设置多少个worker**

Nginx 同 redis 类似都采用了 io 多路复用机制，每个 worker 都是一个独立的进程，但每个进
程里只有一个主线程，通过异步非阻塞的方式来处理请求， 即使是千上万个请求也不在话
下。每个 worker 的线程可以把一个 cpu 的性能发挥到极致。所以 worker 数和服务器的 cpu
数相等是最为适宜的。设少了会浪费 cpu，设多了会造成 cpu 频繁切换上下文带来的损耗。
```txt
# 设置 worker  数量。
worker_processes 4
#work 绑定 cpu(4 work 绑定 4cpu)。
worker_cpu_affinity 0001 0010 0100 1000
#work 绑定 cpu (4 work 绑定 8cpu 中的 4 个) 。
worker_cpu_affinity 0000001 00000010 00000100 00001000
```

**连接数worker_connection**

这个值是表示每个worker进程所能建立连接的最大值，所以，一个 nginx 能建立的最大连接数，应该是 worker_connections \* worker_processes。当然，这里说的是最大连接数，对于HTTP请求 本 地 资 源 来 说 ， 能够支持的最大并发数量是worker_connections \*
worker_processes，如果是支持http1.1的浏览器每次访问要占两个连接，所以普通的静态访问最大并发数是： worker_connections \* worker_processes /2，而如果是HTTP作为反向代理来说，最大并发数量应该是 worker_connections \*
worker_processes/4。因为作为反向代理服务器，每个并发会建立与客户端的连接和与后端服务的连接，会占用两个连接。

{% asset_img 7.png %}

# nginx搭建高可用集群

## Keepalived+Nginx高可用集群（主从模式）

{% asset_img 9.png %}


## Keepalived+Nginx高可用集群（双主模式）

{% asset_img 10.png %}

