---
title: "五分钟搭建容器化Blessing Skin皮肤站"
published: 2024-08-06T23:15:44+08:00
description: "什么人适合看这篇教程？ 有基础Linux知识，会在命令行编辑文件、解决一些基础问题、且对Docker有一定了解的人；若对Docker配置文件和运行机制研究较深可按照本文思路自由修改配置文件"
tags: [Minecraft, Docker, PHP]
category: "教程"
image: ""
draft: false
comment: true
---

:::note[什么人适合看这篇教程？]

有基础Linux知识，会在命令行编辑文件、解决一些基础问题、且对Docker有一定了解的人；若对Docker配置文件和运行机制研究较深可按照本文思路自由修改配置文件

:::

Blessing Skin是一个非常成熟的开源Minecraft皮肤站，使用PHP+Laravel编写，可以在宝塔、1Panel等运维面板服快速部署。但是由于官方并未提供生产环境的Docker镜像（GitHub的那个Dockerfile不适合用于生产环境），所以只能自己编写Dockerfile来实现容器化部署

## 下载blessing-skin-server并配置环境变量

找个文件夹，这里以`/skin`为例，下载并解压blessing-skin-server的最新版release

```
cd / && mkdir skin && cd skin
wget https://github.com/bs-community/blessing-skin-server/releases/download/6.0.2/blessing-skin-server-6.0.2.zip
unzip blessing-skin-server-6.0.2.zip
rm -f blessing-skin-server-6.0.2.zip
```

修改文件夹权限并创建`.env`文件

```
chown -R www-data:www-data /skin
cp .env.example .env
```

编辑`.env`文件

```
vim .env
```

修改redis相关配置并配置插件市场镜像源（官方）

```
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_CLIENT=phpredis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

PLUGINS_REGISTRY=https://d2jw1l0ullrzt6.cloudfront.net/registry_zh_CN.json
```

配置完成后缓存、会话、队列都将被redis接管，提高运行效率

## 编写Dockerfile

然后我们再另外找个文件夹，这里以`/compose`为例

创建并编辑Dockerfile

```
cd /compose
vim Dockerfile
```

将以下内容写入Dockerfile

```
FROM php:8.1-apache

ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

RUN chmod +x /usr/local/bin/install-php-extensions && \
install-php-extensions gd zip pdo_mysql redis

WORKDIR /app

ENV APACHE_DOCUMENT_ROOT /app/public
RUN chown -R www-data:www-data . && \
sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf && \
sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf && \
a2enmod rewrite headers

VOLUME /app

EXPOSE 80
```

:::caution

注意：如果你想使用PostgreSQL，请将`pdo_mysql`改成`pdo_pgsql`

:::

## 编写compose.yaml

还是在`/compose`文件夹，创建并编辑`compose.yaml`

```
services:
  skin:
    build:
      context: .
    volumes:
      - /skin:/app
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "8000:80"
    restart: always
  redis:
    image: redis:alpine
    restart: always
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
```

使用docker compose拉起镜像

```
 docker compose up -d
```

等镜像编译并启动之后访问`localhost:8000`即可

之后可以使用Nginx等Web服务器反代皮肤站，配置https访问
