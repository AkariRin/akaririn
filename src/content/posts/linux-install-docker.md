---
title: "Linux系统快速安装Docker环境"
published: 2024-08-15T19:04:34+08:00
description: "本教程讲解如何使用Docker官方提供的docker-install脚本在Linux系统上快速安装Docker环境"
tags: [Docker, Linux, Debian]
category: "教程"
image: ""
draft: false
comment: true
---

本教程讲解如何使用Docker官方提供的docker-install脚本在Linux系统上快速安装Docker环境

::github{repo="docker/docker-install"}

## 机器位于中国大陆以外区域

若机器能直连`get.docker.com`和`download.docker.com`，则输入以下指令即可完成安装

```
curl -fsSL https://get.docker.com -o install-docker.sh
sudo sh install-docker.sh
```

### 安装特定版本的Docker

使用`--version`参数即可

示例：

```
sudo sh install-docker.sh --version 23.0
```

### 安装测试版Docker

使用`--channel`参数即可

示例：

```
sudo sh install-docker.sh --channel test
```

## 机器位于中国大陆（使用阿里云公网镜像）

若机器无法直连`get.docker.com`和`download.docker.com`，则需要使用`--mirror`参数指定阿里云公网镜像

```
sudo sh install-docker.sh --mirror Aliyun
```

此时Docker安装包将从`https://mirrors.aliyun.com/docker-ce`下载

## 机器位于中国大陆（使用云服务商内网镜像）

若你使用的是阿里云ECS、腾讯云CVM等云服务器或轻量应用服务器，国内大厂一般会有配套的内网镜像

内网镜像的下载速度极高，并且阿里云公网镜像限速30M，相比之下使用内网镜像是一个更好的选择

编辑`install-docker.sh`，在第96行（注：随着Docker官方对脚本的更新，行数可能不再准确）修改`DEFAULT_DOWNLOAD_URL`的值

此处以阿里云内网镜像举例：

```
DEFAULT_DOWNLOAD_URL="http://mirrors.cloud.aliyuncs.com/"
```

注意：阿里云内网使用的是http协议而非https，若使用https会出现异常，具体以阿里云镜像站的说明为准

然后运行脚本即可

```
sudo sh install-docker.sh
```
