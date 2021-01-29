---
title: "安装后的配置信息和命令"
linkTitle: "安装后的配置和命令"
weight: 202
date: 2021-01-24
description: >
  安装Nginx后的基本配置信息和相关命令
---

### 安装的路径概览

安装之后的各种文件路径和文件夹地址：

```bash
$ sudo find / -name nginx
/etc/nginx
/etc/logrotate.d/nginx
/etc/default/nginx         # 目录不存在
/etc/ufw/applications.d/nginx  # 防火墙
/etc/init.d/nginx          # 开机自动启动
/var/log/nginx             # 日志文件路径，包括 access 和 error 日志
/usr/lib/nginx             # 目录不存在
/usr/sbin/nginx				# nginx 二进制文件
/usr/share/doc/nginx       # 文档路径
/usr/share/nginx           # 只有一个 html 文件夹和一个 index.html 文件
```

### 配置目录

在 `/etc/nginx` 目录下的 nginx.conf 文件，默认http配置：

```
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

### 操作命令

nginx相关的操作命令：

```bash
# 查看nginx状态
sudo service status nginx

# nginx操作
sudo systemctl start nginx 
sudo systemctl stop nginx 
sudo systemctl restart nginx
sudo systemctl reload nginx

# 查看错误日志
sudo tail -f /var/log/nginx/error.log
```

