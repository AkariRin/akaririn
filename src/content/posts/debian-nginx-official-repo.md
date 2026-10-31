---
title: "Debian使用Nginx官方源通过包管理器安装Nginx"
published: 2024-08-15T23:29:22+08:00
description: "正常我们通过apt安装nginx的时候，会发现安装的nginx版本比实际发布的最新版release要落后许多个版本，这是因为ubuntu、debian等系统源为了保证稳定性，通常不会直接上传最新版release到他们的源"
tags: [Nginx, apt, Debian]
category: "教程"
image: ""
draft: false
comment: true
---

正常我们通过apt安装nginx的时候，会发现安装的nginx版本比实际发布的最新版release要落后许多个版本，这是因为ubuntu、debian等系统源为了保证稳定性，通常不会直接上传最新版release到他们的源

而网上许多安装教程都是直接编译安装，对于许多低性能机器或者边缘节点十分不友好。不过nginx官方提供了他们自己的apt源以供我们安装最新版nginx

:::note[注意]

Nginx官方文档[https://nginx.org/en/linux_packages.html](https://nginx.org/en/linux_packages.html)也提供了基于yum等其他包管理器的安装方式，鉴于我的服务器都是Debian系统，故这里只展开讲基于apt的安装方式

:::

~~其实本文基本上就是把官方文档翻译了一遍~~:spoiler[（烤肉man）]

#### 准备工作

安装一些要用到的依赖

```
sudo apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
```

#### 下载gpg密钥

下载nginx官方签名密钥用于验证包的真实性

```
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

检查下载的密钥是否正确

```
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

正常输出如下（片段）：

```
pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>
```

正确指纹：`573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62`

注意输出中还包含其他密钥用于对包进行签名

#### 设置apt源

运行以下命令以安装stable版nginx软件源

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

<details>
<summary>运行以下命令以安装mainline版nginx软件源</summary>

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

</details>

:::note[Stable版和Mainline版的区别（来自GPT-4o）]

它们的区别主要在于更新频率和功能特性：

Stable版

- 通常是偶数版本号（例如 1.20.0，1.22.0）
- 更新较少，主要是修复安全漏洞和重大bug，不会频繁引入新功能
- 经过了较长时间的测试和使用，适合生产环境，特别是那些对稳定性要求较高的场景

Mainline版

- 通常是奇数版本号（例如 1.19.0，1.21.0）
- 更新较频繁，会引入新的功能和性能改进
- 虽然也相对稳定，但因为包含了最新的特性和改进，相对来说可能会有更多的bug。适合需要最新功能的用户或开发环境

选择建议：

- 如果你需要一个经过长时间测试并且稳定性非常高的版本用于生产环境，选择Stable版
- 如果你需要最新的功能和改进，并且能接受可能的bug和不稳定性，可以选择Mainline版

:::

设置apt优先级以确保nginx优先从nginx官方源而非系统源（debian官方源）更新

```
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
```

#### 安装Nginx

执行以下指令安装nginx本体

```
sudo apt update
sudo apt install nginx -y
```

此时前往`/etc/nginx/`，可以在这里看到自动生成的nginx配置文件
