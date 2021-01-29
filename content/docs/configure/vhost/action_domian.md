---
title: "基于域名"
linkTitle: "基于域名"
weight: 612
date: 2021-01-18
description: >
  实战：基于域名的虚拟主机
---

## 前言

### 默认包含

默认在 `/etc/nginx/nginx.conf` 配置文件中会有如下配置：

```bash
http {
	......
    ##
    # Virtual Host Configs
    ##

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

这表明默认情况下 nginx 会自动包含 `/etc/nginx/conf.d/*.conf` 和 `/etc/nginx/sites-enabled/*`。

### 启用站点和可用站点

默认情况下，在 `/etc/nginx/sites-enabled` 下有一个默认站点，这个站点也就是 nginx 安装之后的默认站点：

```bash
$ cd /etc/nginx/sites-enabled
$ ls -l
total 0
lrwxrwxrwx 1 root root 34 Oct  6 02:19 default -> /etc/nginx/sites-available/default
```

打开 `/etc/nginx/sites-available/default` 可以看到如下内容：

```bash
server {
        listen 80 default_server;
        listen [::]:80 default_server;

		root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name _;
        location / {
                try_files $uri $uri/ =404;
        }
```

按照这个文档的建议，最好是在 `/etc/nginx/sites-available/` 下建立站点的配置文件，这些站点就是所谓的"可用站点"。然后在 link 到 `/etc/nginx/sites-enabled` 下开启站点，这些开启的站点就是所谓"启用站点"。

通过建立链接来控制可用站点的启用。

## 实战

### 创建虚拟主机 basiccloud.net

目标： http://basiccloud.net 和 http://www.basiccloud.net 应该都指向同一个虚拟主机。

在 `/etc/nginx/sites-available/` 下新建 basiccloud.net 文件，内容如下：

```bash
server {
       listen 80;

       server_name basiccloud.net www.basiccloud.net;

       root /var/www/basiccloud.net;
       index index.html;
}
```

然后建立 `/var/www/basiccloud.net` 目录，准备好站点的html文件。

将 basiccloud.net 站点文件链接到 `/etc/nginx/sites-enabled/` 目录：

```bash
sudo ln -s /etc/nginx/sites-available/basiccloud.net /etc/nginx/sites-enabled/basiccloud.net
```

配置完成之后，在重新转载前，先验证一下：

```bash
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

验证通过，再重新装载：

```bash
sudo nginx -s reload
```

### 创建虚拟主机 dolphin.basiccloud.net

目标： http://dolphin.basiccloud.net 应该指向另外一个虚拟主机。

在 `/etc/nginx/sites-available/` 下新建 dolphin.basiccloud.net 文件，内容如下：

```bash
server {
       listen 80;

       server_name dolphin.basiccloud.net;

       root /var/www/dolphin.basiccloud.net;
       index index.html;
}
```

然后建立 `/var/www/dolphin.basiccloud.net` 目录，准备好站点的html文件。

将 dolphin.basiccloud.net 站点文件链接到 `/etc/nginx/sites-enabled/` 目录：

```bash
sudo ln -s /etc/nginx/sites-available/dolphin.basiccloud.net /etc/nginx/sites-enabled/dolphin.basiccloud.net
```

