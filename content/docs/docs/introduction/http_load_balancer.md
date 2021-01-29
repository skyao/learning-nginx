---
title: "HTTP负载均衡"
linkTitle: "HTTP负载均衡"
weight: 419
description: >
  使用nginx实现HTTP负载均衡
---

> 注：内容翻译自Nginx官网文档 [Using nginx as HTTP load balancer](http://nginx.org/en/docs/http/load_balancing.html)。

## 介绍

在多个应用实例间做负载均衡是一个被广泛使用的技术，用于优化资源效率，最大化吞吐量，减少延迟和容错。

nginx可以作为一个非常高效的HTTP负载均衡器来分发请求到多个应用服务器，并提高web应用的性能，可扩展性和可靠性。

## 负载均衡方法

nginx支持以下负载均衡机制（或者方法）：

- round-robin/轮询： 到应用服务器的请求以round-robin/轮询的方式被分发
- least-connected/最少连接：下一个请求将被分派到活动连接数量最少的服务器
- ip-hash/IP散列： 使用hash算法来决定下一个请求要选择哪个服务器(基于客户端IP地址)

## 默认负载均衡配置(轮询)

nginx中最简单的负载均衡配置看上去大体如下：

```
    http {
        upstream myapp1 {
            server srv1.example.com;
            server srv2.example.com;
            server srv3.example.com;
        }
    
        server {
            listen 80;
    
            location / {
                proxy_pass http://myapp1;
            }
        }
    }
```

在上面的例子中， 同一个应用有3个实例分别运行在srv1-srv3。当没有特别指定负载均衡方法时， 默认为round-robin/轮询。所有请求被代理到服务器集群myapp1， 然后nginx实现HTTP负载均衡来分发请求。

在nginx中反向代理的实现包括HTTP, HTTPS, FastCGI, uwsgi, SCGI, 和 memcached的负载均衡。

要配置负载均衡用HTTPS替代HTTP，只要使用"https"作为协议即可。

为FastCGI, uwsgi, SCGI, 或 memcached 搭建负载均衡时， 只要使用相应的fastcgi_pass, uwsgi_pass, scgi_pass, 和 memcached_pass指令。

## 最少连接负载均衡

另一个负载均衡方式是least-connected/最少连接。当某些请求需要更长时间来完成时，最少连接可以更公平的控制应用实例上的负载。

使用最少连接负载均衡时，nginx试图尽量不给已经很忙的应用服务器增加过度的请求， 而是分配新请求到不是那么忙的服务器实例。

nginx中通过在服务器集群配置中使用least_conn指令来激活最少连接负载均衡方法：

```
    upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

## 会话持久化(ip-hash)

请注意，在轮询和最少连接负载均衡方法中，每个客户端的后续请求被分派到不同的服务器。对于同一个客户端没有任何方式保证发送给同一个服务器。

如果需要将一个客户端绑定给某个特定的应用服务器——用另一句话说，将客户端会话"沾住"或者"持久化"，以便总是能选择特定服务器——，那么可以使用ip-hash负载均衡机制。

使用ip-hash时，客户端IP地址作为hash key使用，用来决策选择服务器集群中的哪个服务器来处理这个客户端的请求。这个方法保证从同一个客户端发起的请求总是定向到同一台服务器，除非服务器不可用。

要配置使用ip-hash负载均衡，只要在服务器集群配置中使用ip_hash指令：

```
    upstream myapp1 {
        ip_hash;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

## 带权重的负载均衡

可以通过使用服务器权重来影响nginx的负载均衡算法。

在上面的例子中，服务器权重没有配置，这意味着所有列出的服务器被认为对于具体的负载均衡方法是完全平等的。

特别是轮询，分派给服务器的请求被认为是大体相当的——假设有足够的请求，并且这些请求被以同样的方式处理而且完成的足够快。

当服务器被指定weight/权重参数时，负载均衡决策会考虑权重。

```
    upstream myapp1 {
        server srv1.example.com weight=3;
        server srv2.example.com;
        server srv3.example.com;
    }
```

With this configuration, every 5 new requests will be distributed across the application instances as the following: 3 requests will be directed to srv1, one request will go to srv2, and another one — to srv3.

在这个配置中，每5个新请求将会如下的在应用实例中分派： 3个请求分派去srv1,一个去srv2,另外一个去srv3.

在最近的nginx版本中，可以类似的在最少连接和IP哈希负载均衡中使用权重。

## 健康检查

nginx中的反向代理实现包含in-band/带内(或者说被动)的服务器健康检查。如果某台服务器响应失败，nginx将标记这台服务器为"失败"，之后的一段时间将尽量避免选择这台服务器来处理后续请求。

max_fails 指令设置在fail_timeout时间内和服务器通讯连续不成功尝试的数量。默认，max_fails设置为0.如果设置为0, 则关闭这台服务器的健康检查。fail_timeout参数同样也定义了服务器被标记为"失败"的时间长度。

在服务器失败之后的fail_timeout间隔时间后， nginx会开始温和的用来自实际客户端的请求来探测服务器。如果探测成功， 服务器将被标记是存活。

## 更多

此外，在nginx中还有更多的指令和参数可以控制服务器负载均衡。例如：proxy_next_upstream, backup, down, 和 keepalive。请查阅我们的参考文档来获取更多信息。

> 注： 后面有翻译的 [HTTP Upstream模块](./upstream_doc.html) 文档。

最后， 应用负载均衡，应用健康检查， 活动监控和on-the-fly 服务器集群重配置在付费的nginx plus中提供。






