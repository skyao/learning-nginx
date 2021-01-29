---
title: "服务器名称"
linkTitle: "服务器名称"
weight: 419
description: >
  Nginx的服务器名称（server name）
---

> 注：内容翻译自Nginx官网文档 [Server Name](http://nginx.org/en/docs/http/server_names.html)。

服务器名称通过使用server_name指令来定义并决定哪个服务器块(server block)将用于给定的请求。参考"[How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)"。

> 注： 中文翻译版本 [nginx如何处理请求](./how_nginx_processes_a_request.html)。

可以使用精确名称，通配符和正则表达式：

```
    server {
        listen       80;
        server_name  example.org  www.example.org;
        ...
    }
    
    server {
        listen       80;
        server_name  *.example.org;
        ...
    }
    
    server {
        listen       80;
        server_name  mail.*;
        ...
    }
    
    server {
        listen       80;
        server_name  ~^(?<user>.+)\.example\.net$;
        ...
    }
```

当通过名称搜索虚拟服务器时， 如果名字和多个指定的变量匹配， 例如同时匹配通配符和正则表达式，在下面的优先级次序中，第一个匹配的变量将被选择：

1. 精确名称
2. 星号开头的最长的通配符名称, 例如 "*.example.org"
3. 星号结束的最长的通配符名称, 例如 "mail.*"
4. 第一个匹配的正则表达式(按照出现在配置文件中的顺序)

# 通配符名称

通配符名称可以在名称的开头和结尾包含星号，并且只能紧挨着点号(.)。名称"www.*.example.org"和"w*.example.org"是不合法的。当然，这些名字可以用正则表达式来指定，例如："~^www\..+\.example\.org$" 和 "~^w.*\.example\.org$". 星号可以匹配多个名称部位，名称 “*.example.org” 不仅可以匹配 www.example.org 还可以匹配 www.sub.example.org.

".example.org"这种特殊的通配符名称可以用于匹配精确名称"example.org"和通配符名称"*.example.org".

# 正则表达式名称

nginx所用的正则表达式兼容于Perl编程语言(PCRE)。为了使用正则表达式， 服务器名称必须以波浪号(~)开头：

	server_name  ~^www\d+\.example\.net$;

否则将被当成是精准名称，或者如果表达式中包含星号就被当成通配符名称(而且大都被认为时不合法).不要忘记设置"^"和"$"锚点。虽然语法上没要求，但是逻辑上需要他们。还要注意域名的点号要使用反斜杠做转义。包含字符"{"和"}"的正则表达式需要使用引号：

	server_name  "~^(?<name>\w\d{1,3}+)\.example\.net$";

否则nginx会启动失败并显示错误信息：

	directive "server_name" is not terminated by ";" in ...

被命名的正则表达式捕获器可以随后作为变量使用：

    server {
        server_name   ~^(www\.)?(?<domain>.+)$;
    
        location / {
            root   /sites/$domain;
        }
    }

PCRE 类库使用下列语法来支持命名捕获器：

    ?<name>			Perl 5.10 兼容语法, 从PCRE-7.0开始支持
    ?'name'			Perl 5.10 兼容语法, 从PCRE-7.0开始支持
    ?P<name>		Python 5.10 兼容语法, 从PCRE-4.0开始支持

# 五花八门的名称

有一些服务器名称需要特别对待。

如果一个非"default"的服务器块需要处理不带"Host" header的请求， 需要指定一个空的名称：

    server {
        listen       80;
        server_name  example.org  www.example.org  "";
        ...
    }

服务器块中如果没有定义server_name，那么nginx将使用空名称作为服务器名。

直到0.8.48版本，nginx在这种情况下使用机器的hostname作为服务器名称。如果服务器名称被定义为"$hostname"(0.9.4), 使用机器的hostname。

如果某些请求使用IP地址替代服务器名称， 请求的"Host" header将包含IP地址， 使用IP地址作为服务器名称可以处理这些请求：

    server {
        listen       80;
        server_name  example.org
                     www.example.org
                     ""
                     192.168.1.1
                     ;
        ...
    }

在这个匹配所有的服务器例子中， 可以看到一个奇怪的名称"_":

    server {
        listen       80  default_server;
        server_name  _;
        return       444;
    }

这个名字没有任何特别之处，它仅仅是无数从来不和实际名称相交的非法域名中的一个。其他非法名称类似"--" 和 "!@#"。

0.6.25版本之前的nginx支持特殊的名称"*"，被错误的理解为是一个匹配所有的名称，但是实际上从来没有工作过。相反，这个功能现在是通过server_name_in_redirect指令来提供的。特殊名称"*"现在被废弃，应该使用server_name_in_redirect指令。注意使用server_name指令时是没有方法可以指定匹配所有的名称或者默认服务器的。这是listen指令的一个属性，而不是server_name指令。请参考[How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)。可以定义服务器监听于端口*:80和*:8080，并指示某个成为端口*:8080的默认服务器，而其他成为端口*:80的默认服务器：

    server {
        listen       80;
        listen       8080  default_server;
        server_name  example.net;
        ...
    }
    
    server {
        listen       80  default_server;
        listen       8080;
        server_name  example.org;
        ...
    }

# 优化

精确名称，以星号开头的通配符名称和以星号结束的通配符名称存储于绑定在监听端口上的三个hash表中。hash表的大小在配置阶段做了优化以便可以以最大的CPU缓存命中来查找服务器。构建hash table的细节在单独的文档中提供。

精确名称的hash表被第一个搜索。如果名称没有找到，继续搜索用星号开头的通配符名称的hash表。如果名字还没有找到，继续搜索用星号结束的通配符名称的hash表。

搜索通配符名称哈希表比搜索精确名称哈希表要慢，因为名称是通过域名部分来搜索的。注意特殊形式的通配符".example.org"是存储在通配符名称哈希表而不是精确名称哈希表。

正则表达式是逐个检验的，因此是最慢的方法，而且不可扩展。

处于这些理由， 最好能尽可能的使用精确名称。例如， 如果一个服务器最频繁的请求名称是example.org和www.example.org， 显式定义他们将更有效率：

    server {
        listen       80;
        server_name  example.org  www.example.org  *.example.org;
        ...
    }
    than to use the simplified form:
    
    server {
        listen       80;
        server_name  .example.org;
        ...
    }

如果定义有大量的服务器名称，或者定义有非正常的长服务器名称， 有必要在http级别调整 server_names_hash_max_size 和 server_names_hash_bucket_size 指令。server_names_hash_bucket_size指令的默认值可能是32,或者64,或者其他值，取决于CPU cache line的大小。如果默认值是32而服务器名被定义为"too.long.server.name.example.org", 那么nginx在启动时会失败并显示错误信息：

    could not build the server_names_hash,
    you should increase server_names_hash_bucket_size: 32

在这种情况下， 指令值应该增加到下一个级别：

    http {
        server_names_hash_bucket_size  64;
        ...

如果定义有大量的服务器名称，会出现另外一个错误：

    could not build the server_names_hash,
    you should increase either server_names_hash_max_size: 512
    or server_names_hash_bucket_size: 32

在这种情况下，先尝试将server_names_hash_max_size设置为接近服务器名称的数量。只有在这种方式无效，或者nginx的启动时间长的不可接收时，尝试增加server_names_hash_bucket_size。

如果某台服务器是该监听接口唯一的服务器，则nginx将完全不检验服务器名字(也不会为监听端口构建哈希表)。但是，有一个例外。如果服务器名称是带有捕获器的正则表达式， 那么nginx将不得不执行表达式以便获取捕获的内容。

# 兼容性

- 特殊服务器名称"$hostname"从0.9.4版本开始支持
- 在0.8.48版本之后，默认服务器名称是空名称""
- 被命名的正则表达式服务器名称捕获器从0.8.25版本开始支持
- 正则表达式服务器名称捕获器从0.7.40版本开始支持
- 空服务器名称""从0.7.12版本开始支持
- 从0.6.25版本开始支持使用通配符服务器名称或者正则表达式名称作为第一个服务器名称
- 从0.6.7版本开始支持使用正则表达式名称
- 从0.6.0版本开始支持"example.*"形式的通配符
- 从0.3.18版本开始支持特别格式.example.org
- 从0.1.13版本开始支持通配符形式*.example.org


































