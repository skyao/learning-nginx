---
title: "HTTP Upstream模块"
linkTitle: "HTTP Upstream模块"
weight: 455
description: >
  Nginx文档中的HTTP Upstream模块
---

> 注：内容翻译自Nginx官网文档 [HTTP Upstream模块](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)，个别地方稍作补充。

ngx_http_upstream_module 模块用于定义可以被proxy_pass, fastcgi_pass, uwsgi_pass, scgi_pass和memcached_pass指令引用的服务器集群。

## 配置范例

```bash
upstream backend {
    server backend1.example.com       weight=5;
    server backend2.example.com:8080;
    server unix:/tmp/backend3;

    server backup1.example.com:8080   backup;
    server backup2.example.com:8080   backup;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

动态可配置集群(Dynamically configurable group)是商业版本([commercial subscription](https://www.nginx.com/products/))的一部分.

```bash
resolver 10.0.0.1;

upstream dynamic {
    zone upstream_dynamic 64k;

    server backend1.example.com      weight=5;
    server backend2.example.com:8080 fail_timeout=5s slow_start=30s;
    server 192.0.2.1                 max_fails=3;
    server backend3.example.com      resolve;

    server backup1.example.com:8080  backup;
    server backup2.example.com:8080  backup;
}

server {
    location / {
        proxy_pass http://dynamic;
        health_check;
    }
}
```

## 指令

### upstream指令

upstream指令用于定义服务器集群。服务器可以监听在不同端口。另外，监听在TCP和UNIX-domain socket的服务器可以混合使用

    Syntax:	upstream name { ... }
    Default:	—
    Context:	http

范例：

```bash
upstream backend {
    server backend1.example.com weight=5;
    server 127.0.0.1:8080       max_fails=3 fail_timeout=30s;
    server unix:/tmp/backend3;

    server backup1.example.com  backup;
}
```

默认，使用带权重的round-robin平衡算法将请求分派到服务器。在上面的例子中， 每7个请求将被如下分配：

- 5个请求去backend1.example.com
- 1个请求去127.0.0.1:8080
- 1个请求去unix:/tmp/backend3

在和某台服务器通讯的过程中，如果发生错误， 请求将被分派给下一个服务器， 以此类推知道所有可用服务器都被尝试。如果没有任何一个服务器可以可以返回成功的应答，则客户端将会收到和最后一台机器的通讯结果。

### server指令

server指令用于定义一台服务器的地址和其他参数。地址可以是域名或者IP地址，端口可选，或者是以"unix:"前缀指定的UNIX-domain socket路径。如果端口没有指定，将使用80端口。可以解析为多个IP地址的域名将一次性定义多台服务器。

    Syntax:	server address [parameters];
    Default:	—
    Context:	upstream

下面是可用的参数列表。

- weight=number

	设置服务器的权重，默认为1.

- max_fails=number

	设置在参数fail_timeout指定的时间内，发生的和服务器通讯的不成功的数量。默认情况，不成功尝试次数被设置为1.如果设置为0则关闭尝试计数。至于什么是不成功的尝试则通过proxy_next_upstream, fastcgi_next_upstream, uwsgi_next_upstream, scgi_next_upstream, 和 memcached_next_upstream指令定义。

- fail_timeout=time

	用于设置：

	- 时间段，在此期间应该发起指定数量的和服务器通讯的不成功尝试，以判断服务器是否不可到达
	- 和接下来服务器被判定为不可到达的时间期间

	默认，fail_timeout参数被设置为10秒钟.

    > 注：如果设置为max_fails=5;fail_timeout=30s，表示如果有5次请求失败，则该服务器被断定为不可到达，之后30s之类将不再尝试这台机器。再之后的每30s，都将进行最多5次尝试，如果继续失败则继续判断为不可到达并不再尝试。

- backup

	标记当前服务器为备用服务器。当主服务器(注：应该是没有标记为backup和down的服务器)都不能达到时请求将被分派过去。

- down

	将当前服务器标记为永久不可到达。

另外，商业版本将支持下面的参数：

- max_conns=number

	限制同时激活的到被代理的服务器的最大连接数。默认值为0,表示没有限制。

	** 当长连接(keepalive connection)和多work被启用时， 到被代理的服务器的最大连接总数可能超过max_conns参数的值**

- resolve

	监视域名对应的服务器IP地址的变化，并自动修改upstream配置而不需要重启nginx服务器(1.5.12版本)。The server group must reside in the shared memory

    > 注：这句不懂....

    为了让这个参数工作，resolver必须指定在http块中。例如：

    ```bash
    http {
        resolver 10.0.0.1;

        upstream u {
            zone ...;
            ...
            server example.com resolve;
        }
    }
    ```

- route=string

	设置服务器路由名字(route name)

- slow_start=time

	设置时间段，在此期间服务器将从0到正常值逐渐恢复它的权重，当服务器从不健康变成健康，或者服务器从被标记为不可到达一段时间后变成可到达时。默认值是0, 表示关闭缓慢启动功能。

** 如果服务器集群中仅有一台服务器， max_fails, fail_timeout 和 slow_start 参数都将被忽略， 而这台机器永远不会被标记为不可到达。**

## zone指令(商业版)

zone指令在版本1.9.0中出现。

	Syntax:	zone name [size];
	Default:	—
	Context:	upstream

定义共享内存zone的名称和大小， 共享内存zone用于保存集群配置和运行时状态，以便在worker进程之间共享。多个集群可以分享同一个zone。在这种情况下，只需要指定一次大小。

另外，作为商业版本的一部分，这样的集群容许改变集群成员或者修改某台服务器的设置而不需要重启nginx。

### state指令(商业版)

state指令在版本1.9.7中出现，属于商业版本的一部分.

    Syntax:	state file;
    Default:	—
    Context:	upstream

指定一个文件来保存动态可配置集群的状态。状态目前仅限于服务器列表及其参数.当解析配置时state文件被读取并在每次upstream配置被修改时更新。直接修改这个文件的内容是无效的。这个指令不能server指令一起使用。

configuration reload 或者 binary upgrade造成的修改可能丢失。

### hash指令

hash指令在版本1.7.2中出现。

    Syntax:	hash key [consistent];
    Default:	—
    Context:	upstream

指定服务器集群的负载均衡方法，客户端-服务器映射基于散列key值。key可以包含文本，变量和他们的组合。注意从集群中添加或者删除一个服务器可能导致大部分key重新映射到不同的服务器。这个方法兼容Cache::Memcached Perl类库。

如果consistent参数被指定，则将使用ketama一致性哈希算法。这个算法确保当有一台服务器添加到集群或者从集群中删除时仅有少数key被重新映射到不同的服务器。这将有助于缓存系统实现更高的缓存命中率。ketama_points参数设置为160时，这个方法兼容Cache::Memcached::Fast Perl 类库。

### ip_hash指令

    Syntax:	ip_hash;
    Default:	—
    Context:	upstream

指定集群使用的负载均衡算法，基于客户端IP地址将请求分派给服务器。客户端IPv4地址的前三个 八位字节，或者整个IPv6地址被作为哈希key来使用。这个算法保证从同一个客户端来的请求总是被分派到同样的服务器，除非这个服务器不可达到。后面这种情况下客户端请求将被分派到其他服务器。大多数情况，请求总是被分派到同一个服务器。

IPv6地址的支持是从版本1.3.2 和 1.2.2开始。

如果某台服务器需要被永久移除，那么应该将它标记为down以便保持当前的客户端IP地址的哈希。

例如:

```bash
upstream backend {
    ip_hash;

    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
    server backend4.example.com;
}
```

在版本1.3.1 和 1.2.2之前，无法指定使用ip_hash负载均衡算法的服务器的权重。

> 注：言下之意，这两个版本之后就可以指定权重了？）

## keepalive指令

keepalive指令出现在版本1.1.4。

    Syntax:	keepalive connections;
    Default:	—
    Context:	upstream

激活到upstream服务器的连接缓存（注：即长连接）。

connections参数设置每个worker进程在缓冲中保持的到upstream服务器的**空闲keepalive连接的最大数量**.当这个数量被突破时，最近使用最少的连接将被关闭。

特别提醒：keepalive指令**不会**限制一个nginx worker进程到upstream服务器连接的总数量。connections参数应该设置为一个足够小的数字来让upstream服务器来处理新进来的连接。

> 注： 这句话的语义非常的费解，原文如下：
>
> The connections parameter should be set to a number small enough to let upstream servers process new incoming connections as well.
> 我的理解是如果想让upstream每次都处理新的进来的连接，就应该将这个值放的足够小。反过来理解，就是如果不想让upstream服务器处理新连接，就应该放大一些？

使用keepalive连接的memcached upstream配置的例子：

```bash
upstream memcached_backend {
    server 127.0.0.1:11211;
    server 10.0.0.2:11211;

    keepalive 32;
}

server {
    ...

    location /memcached/ {
        set $memcached_key $uri;
        memcached_pass memcached_backend;
    }
}
```

对于HTTP，proxy_http_version指定应该设置为"1.1"，而"Connection" header应该被清理：

```bash
upstream http_backend {
    server 127.0.0.1:8080;

    keepalive 16;
}

server {
    ...

    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```

或者，HTTP/1.0 持久连接可以通过传递"Connection: Keep-Alive" header 到upstream server， 但是不推荐使用这种方法。

对于FastCGI服务器，要求设置fastcgi_keep_conn来让长连接工作：

```bash
upstream fastcgi_backend {
    server 127.0.0.1:9000;

    keepalive 8;
}

server {
    ...

    location /fastcgi/ {
        fastcgi_pass fastcgi_backend;
        fastcgi_keep_conn on;
        ...
    }
}
```

当使用默认的round-robin之外的负载均衡算法时，必须在keepalive指令之前激活他们。

__SCGI 和 uwsgi 协议没有keepalive连接的概念。__

## ntlm指令(商业版)

ntlm指令出现在版本 1.9.2 中。

    Syntax:	ntlm;
    Default:	—
    Context:	upstream

容许使用NTLM算法代理请求。一旦客户端发送一个带有"Authorization" header 并且值是"Negotiate" 或 "NTLM"开头时，upstream 连接被绑定到这个客户端连接。后续的客户端请求将被通过同样一个upstream连接代理，以保持认证的上下文。

为了让NTLM算法工作，必须开启到upstream服务器的keepalive连接。proxy_http_version指定应该设置为"1.1"，而"Connection" header应该被清理：

```bash
upstream http_backend {
    server 127.0.0.1:8080;

    ntlm;
}

server {
    ...

    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```

当使用默认的round-robin之外的负载均衡算法时，必须在ntlm指令之前激活他们。

__ntlm指令是商业版本的一部分。__

## least_conn指令

least_conn指令出现于版本 1.3.1 和 1.2.2.

    Syntax:	least_conn;
    Default:	—
    Context:	upstream

指定集群应该使用的负载均衡方法，分派请求到活动连接数量最少的服务器， 兼顾服务器权重。如果有多台这样的服务器，这些服务器将尝试轮流使用带权重的round-robin平衡算法。

> 这句话不太理解，原文：
> If there are several such servers, they are tried in turn using a weighted round-robin balancing method.
> 我的理解，如果有多台服务器的连接数都相同，都是最小，这是这几台服务器之间选择的算法就用带权重的round-robin？

## least_time指令(商业版)

least_time指令出现于把版本1.7.10.

    Syntax:	least_time header | last_byte;
    Default:	—
    Context:	upstream

指定集群应该使用的负载均衡方法，分派请求到平均响应时间最快和活动连接数量最少的服务器， 兼顾服务器权重。如果有多台这样的服务器，这些服务器将尝试轮流使用带权重的round-robin平衡算法。

如果header参数被指定，使用接收到响应header的时间;如果last_byte参数被指定，使用接收到完整响应的时间。

__least_time指令是商业版本的一部分。__

## health_check指令(商业版)

开启对所在location引用的集群中的服务器的定期健康检查。

    Syntax:	health_check [parameters];
    Default:	—
    Context:	location

下面可选参数将被支持：

- interval=time

	设置两次连续健康检查之间的间隔时间，默认5秒钟。

- fails=number

	设置连续失败的健康检查次数，之后这台服务器会被认为是不健康，默认为1.

- passes=number

	设置连续通过的健康检查次数，之后这台服务器会被认为是健康的，默认为1.

- uri=uri

	定义健康检查请求的URL，默认是"/".

- match=name

	指定匹配块来配置测试可以通过健康检查的响应。默认，响应的状态码应该是2xx 或者 3xx;

- port=number

	定义到执行健康检查的服务器的连接端口(1.9.7)。默认，和服务器端口相同。

例如：

```bash
location / {
    proxy_pass http://backend;
    health_check;
}
```

将每5秒钟发送"/"请求给后端集群中的每台服务器。如果发生任何通讯错误或者超时，或者被代理的服务器返回的状态码不是2xx 或者 3xx，则健康检查失败，服务器将被认为是不健康。客户端请求不会发送给不健康的服务器。

健康检查可以配置为检验响应的状态码，是否存在特定header和他们的值，还有http body内容。检验可以使用match指令单独配置并在match参数中引用。例如：

```bash
http {
    server {
    ...
        location / {
            proxy_pass http://backend;
            health_check match=welcome;
        }
    }

    match welcome {
        status 200;
        header Content-Type = text/html;
        body ~ "Welcome to nginx!";
    }
}
```

这个配置表明，为了通过健康检查，健康检查请求的应答应该成功，状态是200, content type 是 "text/html"，并在body中包含"Welcome to nginx!".

The server group must reside in the shared memory.

如果同一个集群的服务器定义有多个健康检查，任何一个检查失败都会导致对应的服务器被认为是不健康。

请注意：当被用于健康检查时，大部分变量都将是空值。

__health_check指令是商业版本的一部分。__

## match指令(商业版)

    Syntax:	match name { ... }
    Default:	—
    Context:	http

定义命名的检验集合用于验证健康检查请求的应答。

下面的选项可以在应答中检验：

- status 200;

	状态码是 200

- status ! 500;

	状态码不是 500

- status 200 204;

	状态码是 200 或 204

- status ! 301 302;

	状态码是 301 或 302

- status 200-399;

	状态码在 200 到 399 的范围内

- status ! 400-599;

	状态码不在 400 到 599 的范围内

- status 301-303 307;

	状态码是 301, 302, 303, 或 307

- header Content-Type = text/html;

	header包含“Content-Type” 并且值是 text/html

- header Content-Type != text/html;

	header包含“Content-Type” 并且值不是 text/html

- header Connection ~ close;

	header包含“Connection” 并且值匹配正则表达式 close

- header Connection !~ close;

	header包含“Connection” 并且值不匹配正则表达式 close

- header Host;

	header包含“Host”

- header ! X-Accel-Redirect;

	header不带 “X-Accel-Redirect”

- body ~ "Welcome to nginx!";

	body 匹配正则表达式 “Welcome to nginx!”

- body !~ "Welcome to nginx!";

	body 不匹配正则表达式 “Welcome to nginx!”

如果指定有多个检验，响应必须配置所有检验。

**仅检查响应body的前256K**

例如:

```bash
## status is 200, content type is "text/html",
# and body contains "Welcome to nginx!"
match welcome {
    status 200;
    header Content-Type = text/html;
    body ~ "Welcome to nginx!";
}
## status is not one of 301, 302, 303, or 307, and header does not have "Refresh:"
match not_redirect {
    status ! 301-303 307;
    header ! Refresh;
}
# status ok and not in maintenance mode
match server_ok {
    status 200-399;
    body !~ "maintenance mode";
}
```

__match指令是商业版本的一部分。__

## queue指令(商业版)

queue指令出现于版本1.5.12.

    Syntax:	queue number [timeout=time];
    Default:	—
    Context:	upstream

当处理请求时，如果某台upstream服务器无法立即被选择，并且集群中的其他服务器都已经达到了max_conns限制，请求将被放置在queue中。如果queue满了，或者要分派请求的服务器无法在timeout参数所设置的时间内选择请求，将会给客户端返回502(Bad Gateway)错误。

timeout参数的默认值是60秒。

__queue指令是商业版本的一部分。__

## sticky指令(商业版)

sticky指令出现于版本1.5.7.

    Syntax:
    	sticky cookie name [expires=time] [domain=domain] [httponly] [secure] [path=path];
    	sticky route $variable ...;
    	sticky learn create=$variable lookup=$variable zone=name:size [timeout=time];
    Default:	—
    Context:	upstream

开启session affinity(注：翻译为会话保持？)特性， 从同样的客户端发出的请求被分派到服务器集群中同样的服务器上。有三个方法可选：

### cookie

当使用cookie方法时， 被指派的服务器信息将在nginx生成的HTTP cookie中传递：

    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
    
        sticky cookie srv_id expires=1h domain=.example.com path=/;
    }

如果客户端还没有被绑定到特定的服务器，请求进来时将被分派到配置的均衡算法选择的服务器。后续带有cookie的请求将被分派给指派的服务器。如果指派的服务器无法处理请求，新的服务器将被选择，就如同客户端没有被绑定一样。

第一个参数设置cookie的名字。其他参数如下：

- expires=time

	设置浏览器保存cookie的时间。特殊值"max"将设置cookie过期时间为"31 Dec 2037 23:55:55 GMT".如果这个参数没有指定， cookie将在浏览器session结束时过期。

- domain=domain

	定义cookie设置的domain

- httponly

	添加HttpOnly属性到cookie(1.7.11)

- secure

	添加Secure属性到cookie(1.7.11)

- path=path

	定义cookie设置的path属性。

__以上任何一个参数，如果缺失则对应的cookie就不会被设置__

### route

当使用route方法时，代理服务器基于第一次请求的接收情况给客户端分派一个路由(route)。这个客户端随后的所有请求将在cookie或者URI中携带路由信息。这些信息类似于server指令中的route参数。如果被指派的服务器无法处理请求，新的服务器将被选择，就如同请求中没有route信息一样

route方法的参数知名可能包含路由信息的变量。第一个非空变量将用于查找匹配的服务器。

例如:

    map $cookie_jsessionid $route_cookie {
        ~.+\.(?P<route>\w+)$ $route;
    }
    
    map $request_uri $route_uri {
        ~jsessionid=.+\.(?P<route>\w+)$ $route;
    }
    
    upstream backend {
        server backend1.example.com route=a;
        server backend2.example.com route=b;
    
        sticky route $route_cookie $route_uri;
    }

这里，如果请求中存在"JSESSIONID" cookie，则从"JSESSIONID" cookie中获取到路由信息。否在，从URI中取。

### learn

When the learn method (1.7.1) is used, nginx analyzes upstream server responses and learns server-initiated sessions usually passed in an HTTP cookie.

当使用learn方法时，nginx分析upstream服务器的响应并学习通常在HTTP cookie中传递的server-initiated 会话。

    upstream backend {
       server backend1.example.com:8080;
       server backend2.example.com:8081;
    
       sticky learn
              create=$upstream_cookie_examplecookie
              lookup=$cookie_examplecookie
              zone=client_sessions:1m;
    }

在这个例子中， upstream server通过在应答中设置cookie "EXAMPLECOOKIE"创建会话。带有这个cookie的后续请求将被分派到同一个服务器。如果服务器不能处理请求，新的服务器将被选择，就如同客户端没有被绑定一样。

参数create 和 lookup 分别指定变量来指示如何创建新会话和搜索已经存在的会话。两个参数都可以指定多个，这样第一个非空的变量将被使用。

会话存储在shared memory zone，名字和大小在zone属性中配置。在64位平台上一个megabyte zone可以存储大概8000个会话。在timeout参数指定的期间内没有被访问的会话将被从zone上移除。默认，超时时间设置为10分钟。

__sticky指令是商业版本的一部分。__

## sticky_cookie_insert指令(废弃)

    Syntax:	sticky_cookie_insert name [expires=time] [domain=domain] [path=path];
    Default:	—
    Context:	upstream

sticky_cookie_insert指令在版本1.5.7之后就被废弃了。被sticky指令替代。

