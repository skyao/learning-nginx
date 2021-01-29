---
title: "nginx如何处理请求"
linkTitle: "nginx如何处理请求"
weight: 419
description: >
  nginx如何处理请求
---

> 注：内容翻译自Nginx官网文档 [How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)。

# 基于名称的虚拟服务器

nginx首先要决定哪个服务器应该处理请求。让我们从一个简单的配置开始，三个虚拟服务器都监听在端口*:80:

    server {
        listen      80;
        server_name example.org www.example.org;
        ...
    }
    
    server {
        listen      80;
        server_name example.net www.example.net;
        ...
    }
    
    server {
        listen      80;
        server_name example.com www.example.com;
        ...
    }

在这个配置中，nginx仅仅检验请求header中的"Host"域来决定请求应该路由到哪个服务器。如果它的值不能匹配任何服务器，或者请求完全没有包含这个header域，那么nginx将把这个请求路由到这个端口的默认服务器。在上面的配置中，默认服务器是第一个 - 这是nginx标准的默认行为。也可以通过listen指令的default_server属性来显式的设置默认服务器:

	server {
	    listen      80 default_server;
	    server_name example.net www.example.net;
	    ...
	}

> default_server 参数从版本0.8.21开始可用，在更早的版本中要使用default参数。

注意默认服务器是监听端口的一个属性，而不是服务器名称。后面会有更多描述。

# 如何防止使用未定义的服务器名称来处理请求

如果容许请求没有"Host" header 域，放弃这些请求的服务器可以定义为：

    server {
        listen      80;
        server_name "";
        return      444;
    }

这里，服务器名称被设置为空字符串，这样将匹配没有"Host"header域的请求， 并返回一个特殊的nginx的非标准码404，然后关闭连接。

> 从版本0.8.48开始，这是服务器名称的默认设置， 因此server_name ""可以不用写。在更早的版本中，机器的hostname被用作默认服务器名称。

# 基于名称和基于IP混合的虚拟服务器

让我们看一下更复杂的配置，有一些虚拟服务器监听在不同的地址：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }
    
    server {
        listen      192.168.1.1:80;
        server_name example.net www.example.net;
        ...
    }
    
    server {
        listen      192.168.1.2:80;
        server_name example.com www.example.com;
        ...
    }

在这个配置中，nginx首先通过server块的listen指令检验请求的IP地址和端口。然后在通过server块的server_name入口检验请求的"Host"header域。如果服务器名称没有找到，请求将被默认服务器处理。例如，在端口192.168.1.1:80接收到的去www.example.com的请求将被端口192.168.1.1:80的默认服务器处理。这里是第一个服务器，因为这个端口没有定义www.example.com。

前面已经提到，默认服务器是监听端口的属性，并且不同的端口可以定义不同的默认服务器：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }
    
    server {
        listen      192.168.1.1:80 default_server;
        server_name example.net www.example.net;
        ...
    }
    
    server {
        listen      192.168.1.2:80 default_server;
        server_name example.com www.example.com;
        ...
    }

# 一个简单的PHP站点配置

现在让我们看一下nginx如何选择location来为典型而简单的PHP站点处理请求：

    server {
        listen      80;
        server_name example.org www.example.org;
        root        /data/www;
    
        location / {
            index   index.html index.php;
        }
    
        location ~* \.(gif|jpg|png)$ {
            expires 30d;
        }
    
        location ~ \.php$ {
            fastcgi_pass  localhost:9000;
            fastcgi_param SCRIPT_FILENAME
                          $document_root$fastcgi_script_name;
            include       fastcgi_params;
        }
    }

nginx首先搜索由书面字符串给定的最为特别的前缀location，无视列表顺序。在上面的配置中仅有一个带前缀的location "/"，并且因为它匹配任何请求，它将用于作为最后的对策。然后nginx检查通过正则表达式给定的location，基于在配置文件中列出的顺序。第一个匹配的表达式将停止搜索然后nginx将使用这个location。如果没有正则表达式匹配请求，则nginx将使用前面发现的最为特别的前缀location。

注意所有类型的location仅仅检验请求行(HTTP中的request line)中的URL部分，不带参数。这是因为请求字符串中的参数可以以多种方式给出，例如：

    /index.php?user=john&page=1
    /index.php?page=1&user=john

还有，任何人都可能使用这样的查询字符串来请求：

	/index.php?page=1&something+else&user=john

现在让我们来看在上面的配置中请求将如何被处理：

- 请求“/logo.gif” 首先匹配前缀location “/” 然后匹配正则表达式“\.(gif|jpg|png)$”, 因此, 它被后面的location处理。使用指令“root /data/www”，请求被映射到文件/data/www/logo.gif, 然后文件被发送到客户端。
- 请求 “/index.php” 同样首先匹配到前缀location “/” 然后匹配正则表达式“\.(php)$”. 因此, 它被后面的location处理， 请求被分派到监听在localhost:9000的FastCGI服务器。 fastcgi_param 指令设置FastCGI 参数 SCRIPT_FILENAME 为 “/data/www/index.php”, 然后FastCGI 服务器执行这个文件. 变量 $document_root 等同于root指令的值，而变量$fastcgi_script_name 等同于请求 URI, 例如 “/index.php”.
- 请求 “/about.html” 仅仅匹配前缀location“/”, 因此, 它在这个location中被处理. 通过指令 “root /data/www” 请求被映射为文件/data/www/about.html, 然后这个文件被发送到客户端。
- 处理请求 “/”要更复杂一些. 它仅仅匹配前缀location“/”, 因此, 它在这个location中被处理. 然后index指令根据它的参数和“root /data/www”指令来检验index文件的存在。如果文件/data/www/index.html不存在，而文件 /data/www/index.php 存在, 则指令执行一个内部重定向到“/index.php”, 然后 nginx 重新搜索location就像这个请求是从客户端发过来一样。如我们前面所见，这个重定向请求最终将被FastCGI服务器处理。
