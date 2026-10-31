---
title: "没有测试名额也能玩《明日方舟：终末地》二测"
published: 2025-02-17T10:51:58+08:00
description: "这次的二测相比于一测很多细节都优化了许多，但是鉴于测试名额太少，许多人只能干瞪眼看着别人玩自己却玩不上，本文介绍通过搭建本地服务端的方式来体验终末地的教程"
tags: [明日方舟, 终末地, mitmproxy, MongoDB]
category: "教程"
image: "./images/endfield-beta-local-server-1.png"
draft: false
comment: true
---

这次的二测相比于一测很多细节都优化了许多，但是鉴于测试名额太少，许多人只能干瞪眼看着别人玩自己却玩不上，本文介绍通过搭建本地服务端的方式来体验终末地的教程

:::caution[注意]

二月底官方启动器内已经关闭游戏本体下载，请自行寻找游戏本体的存档

:::

:::caution[注意]

第三方服务端仅供学习交流使用，严禁用于商业用途，请于下载后24小时内删除

:::

:::note

此服务端中的抽卡与干员培养资源数量均锁定在一千万，并且没有线性新手教程等内容，仅供体验，想要完整的游戏体验还是期待一下正式版上线吧

:::

<details>
<summary>游戏内截图欣赏</summary>

![](./images/endfield-beta-local-server-1.png)

![](./images/endfield-beta-local-server-2.png)

![](./images/endfield-beta-local-server-3.png)

</details>

## 下载启动器与游戏本体

点击[这里](https://launcher.hg-cdn.com/uXUuLlNbIYmMMTlN/launcher/6/1/Windows/0.3.8_Sf0rvOfH0nZWp5XU/Endfield_TBeta_global_setup.exe)从鹰角官方CDN下载游戏启动器，安装完成后等待启动器自身更新与游戏本体的下载安装

:::note

鉴于游戏本体的下载时间较长，建议在下载游戏本体的同时进行下面服务端的相关配置步骤

:::

## 下载与配置MongoDB

点击[这里](https://fastdl.mongodb.org/windows/mongodb-windows-x86_64-8.0.4.zip)从MongoDB官网下载MongoDB二进制文件压缩包

解压后文件夹内应有如下内容：

![](./images/endfield-beta-local-server-4.png)

在同一目录下创建`data`文件夹用于存放数据库的数据文件

创建`start.cmd`并写入如下内容作为数据库启动脚本：

```
mongod.exe --dbpath data
```

运行启动脚本，若终端输出类似于这样，则代表数据库运行正常：

![](./images/endfield-beta-local-server-5.png)

## 下载与配置服务端

前往该项目下载最新版release的windows版服务端并解压

::github{repo="SuikoAkari/ArkFieldPS"}

下载该项目并将`Json`和`TableCfg`文件夹放入`ArkFieldPS.exe`所在的目录

::github{repo="PotRooms/EndFieldData"}

运行`ArkFieldPS.exe`，出现以下输出内容则代表启动成功：

![](./images/endfield-beta-local-server-6.png)

## 下载与配置mitmproxy

点击[这里](https://downloads.mitmproxy.org/11.1.2/mitmproxy-11.1.2-windows-x86_64.zip)下载mitmproxy并解压

### 安装证书

运行一遍`mitmproxy.exe`后直接关闭

在`用户目录\.mitmproxy`下（完整路径类似于`C:\Users\你的用户名\.mitmproxy`），找到`mitmproxy-ca.p12`并运行，会弹出一个证书导入向导，除了`证书存储`选择`受信任的根证书颁发机构`，其他的一律使用默认设置即可

![](./images/endfield-beta-local-server-7.png)

### 创建路由脚本

在mitmproxy目录下创建`proxy.cmd`并写入如下内容：

```
mitmproxy -s ak.py
```

在同一目录下创建`ak.py`并写入如下内容：

```
import mitmproxy
from mitmproxy import ctx, http

class EndFieldModifier:
    def requestheaders(self, flow: mitmproxy.http.HTTPFlow):
        if "gryphline.com" in flow.request.host or "hg-cdn.com" in flow.request.host:
            if flow.request.method == "CONNECT":
                return

            flow.request.scheme = "http"
            flow.request.cookies.update({
                "OriginalHost": flow.request.host,
                "OriginalUrl": flow.request.url
            })
            flow.request.host = "localhost"
            flow.request.port = 5000
            ctx.log.info("URL:" + flow.request.url)

addons = [
    EndFieldModifier()
]
```

运行`proxy.cmd`，若命令行左下角有`[scripts:1]`则代表脚本已成功加载：

![](./images/endfield-beta-local-server-8.png)

### 设置系统代理

在`设置 - 网络和Internet - 代理`设置中，将`使用代理服务器`调成`开`，并将`代理IP地址`和`端口`分别设置为`127.0.0.1`和`8080`

:::caution[注意]

修改该设置后可能会导致梯子等代理软件失效，建议在运行mitmproxy的时候关闭梯子或加速器等代理软件

:::

---

至此，你可以启动终末地客户端享受游戏了

## 再次游玩终末地要启动哪些东西

之后每次启动客户端之前，你依次前往`MongoDB`文件夹运行`start.cmd`、`EndFieldPS`文件夹运行`EndFieldPS.exe`、`mitmproxy`文件夹运行`proxy.cmd`，当这三个程序都正常启动后即可开始游玩终末地
