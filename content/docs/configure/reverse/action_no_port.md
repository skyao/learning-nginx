---
title: "域名免端口"
linkTitle: "域名免端口"
weight: 622
date: 2021-01-18
description: >
  实战：域名免端口
---

## 前言

在实际使用中，由于web服务器启动于不同进程，因此需要指定不同的端口，也就意味着必然有web应用要使用80之外的端口，这样在地址栏中就必须出现端口号，非常影响用户体验。

比较好的方式，通过使用不同的域名或者二级域名，然后通过nginx反向代理的方式转发请求给到实际负责处理的服务器。

下面是这种方式的典型使用场景的例子：

| 实际web服务器的地址 | 反向代理之后给到终端用户的地址 |
|--------|--------|
|    *：8800    |   http://git.basiccloud.net     |
|    *：8081    |   http://maven.basiccloud.net     |

## 实战

### 创建虚拟主机 git.basiccloud.net

目标：http://git.basiccloud.net 应该指向当前机器上运行于 8800 端口的 gitlab 服务器。

在 `/etc/nginx/sites-available/` 下新建 git.basiccloud.net 文件，内容如下：

```bash
server {
       listen 80;

       server_name git.basiccloud.net;

       location /
       {
              proxy_pass http://127.0.0.1:8800;
       }
}
```

将 git.basiccloud.net 站点文件链接到 `/etc/nginx/sites-enabled/` 目录：

```bash
sudo ln -s /etc/nginx/sites-available/git.basiccloud.net /etc/nginx/sites-enabled/git.basiccloud.net
```

修改完成之后，使用命令检测配置修改结果并重新装载配置：

```bash
sudo nginx -t
sudo nginx -s reload
```

### 创建虚拟主机 maven.basiccloud.net

目标：http://maven.basiccloud.net 应该指向当前机器上运行于 8081 端口的 artifactory 服务器。

在 `/etc/nginx/sites-available/` 下新建 maven.basiccloud.net 文件，内容如下：

```bash
server {
       listen 80;

       server_name maven.basiccloud.net;

       location /
       {
              proxy_pass http://127.0.0.1:8081;
       }
}
```

将 maven.basiccloud.net 站点文件链接到 `/etc/nginx/sites-enabled/` 目录：

```bash
sudo ln -s /etc/nginx/sites-available/maven.basiccloud.net /etc/nginx/sites-enabled/maven.basiccloud.net
```
