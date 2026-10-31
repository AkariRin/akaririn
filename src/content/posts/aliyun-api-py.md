---
title: "一个Python阿里云API封装库"
published: 2024-08-07T17:15:13+08:00
description: "最近有一个脚本需要调用阿里云API自动重启ECS。最开始的想法是使用Python SDK，但是pip install之后看到那一坨示例代码都不知道怎么形容。遂决定自己构建请求"
tags: [阿里云, Python, SDK]
category: "项目"
image: ""
draft: false
comment: true
---

最近有一个脚本需要调用阿里云API自动重启ECS。最开始的想法是使用Python SDK，但是pip install之后看到那一坨示例代码都不知道怎么形容。遂决定自己构建请求

不过幸运的是V3版本请求体&签名机制[https://help.aliyun.com/zh/sdk/product-overview/v3-request-structure-and-signature](https://help.aliyun.com/zh/sdk/product-overview/v3-request-structure-and-signature)页面给出的示例相对比较干净，因此自己照葫芦画瓢封装了一下并发布了一个PyPI包

不过这个包不算是SDK，因为没有把每个API都封装成函数~~（那样太累了）~~，只封装了最基础的调用API的功能

::github{repo="AkariRin/aliyun-api-py"}
