---
title: "记录：迁移到新服务器"
linkTitle: "迁移到新服务器"
weight: 209
date: 2021-01-24
description: >
  记录将Nginx上的站点迁移到新服务器的过程（备忘）
---

原服务器到期，更换服务器之后，需要将原有nginx下的部署的网站迁移到新的服务器上。

## 迁移数据到新服务器

数据包括网站文件和相关的nginx配置。

### 迁移sites-available目录

```bash
cd
mkdir -p migrate/nginx/
scp -r sky@119.28.182.148:/etc/nginx/sites-available/ /home/sky/migrate/nginx/
cd migrate/nginx/sites-available/ 
rm -r default
sudo cp -r * /etc/nginx/sites-available/
```

### 迁移sites-enabled目录

这个目录下都是软链接：

```bash
ls -l
lrwxrwxrwx 1 root root 34 Jan  6  2018 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 41 Jan 27  2018 docs.dreamfly.io.https -> ../sites-available/docs.dreamfly.io.https
lrwxrwxrwx 1 root root 30 Jan 21  2018 dreamfly.io -> ../sites-available/dreamfly.io
lrwxrwxrwx 1 root root 36 Jan 24  2018 dreamfly.io.https -> ../sites-available/dreamfly.io.https
lrwxrwxrwx 1 root root 38 Jan  6  2018 jenkins.dreamfly.io -> ../sites-available/jenkins.dreamfly.io
lrwxrwxrwx 1 root root 27 Jan  7  2018 skyao.io -> ../sites-available/skyao.io
lrwxrwxrwx 1 root root 33 Jan 24  2018 skyao.io.https -> ../sites-available/skyao.io.https
```

通过下面这个特殊的cp命令可以为这些文件创建软链接：

```
cd /etc/nginx/sites-enabled
sudo cp -rs /etc/nginx/sites-available/* .
```

### 迁移 skyao 网站内容

原有网站内容存放在 `/var/www/skyao` 下，直接tar一个包：

```bash
cd /var/www
sudo tar cfv skyao.tar skyao/
```

然后通过scp复制到新的站点，解压缩到 `/var/www` 下：

```bash
cd migrate/nginx/
scp sky@119.28.182.148:/var/www/skyao.tar /home/sky/migrate/
tar xvf skyao.tar
sudo mv skyao /var/www
```

### 迁移部署脚本

`/var/www/`下还有一些部署脚本，一起复制过去：

```bash
sudo scp sky@119.28.182.148:/var/www/*.sh /var/www
```

## 在新服务器上部署站点

直接启动nginx，发现报错，原因是开始了 https 但是新服务器上没有对应的证书。

证书就不迁移了，重新生成一遍。

### 重新生成letsencrypt证书

```
sudo letsencrypt certonly --webroot -w /var/www/skyao -d skyao.io

Challenge failed for domain skyao.io

To fix these errors, please make sure that your domain name was
   entered correctly and the DNS A/AAAA record(s) for that domain
   contain(s) the right IP address.
```

解决方案：

1. 修改DNS解析，让 skyao.io 域名指向新的服务器IP
2. 恢复sky.io 网站的非加密访问，以便通过 letsencrypt 的验证。

### 启动非加密网站

修改 `/etc/nginx/sites-available/skyao.io`，内容修改为下面的内容：

```
server {
   listen 80;

   server_name skyao.io www.skyao.io;
   root /var/www/skyao;

   index index.html index.htm index.nginx-debian.html;
}
```

暂时移除https站点。执行 `sudo systemctl restart nginx` 重启nginx，通过浏览器访问 http://skyao.io 正常之后，再继续：

```
sudo letsencrypt certonly --webroot -w /var/www/skyao -d skyao.io
```

这次就可以正常生成证书了。

### 重新开启https加密

修改 `/etc/nginx/sites-available/skyao.io`，内容修改为下面的内容：

```
server {
   listen 80;

   server_name skyao.io www.skyao.io;
   rewrite ^(.*)$  https://$host$1 permanent;
}
```

然后enable https站点，再执行 `sudo systemctl restart nginx` 重启nginx，即可恢复https站点。

