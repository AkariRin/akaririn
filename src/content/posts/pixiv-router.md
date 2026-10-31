---
title: "【炒冷饭】Pixiv导航工具"
published: 2024-08-12T00:40:50+08:00
description: "鉴于老博客的数据全部丢失了，这里重新写一下以前写过的小项目介绍 这是一个输入PID、作者ID、tag就可以自动跳转到pixiv官网的对应位置的纯静态网页，基于Vue2+Vuetify（上古技术栈）"
tags: [Vue, Vuetify]
category: "项目"
image: ""
draft: false
comment: true
---

~~鉴于老博客的数据全部丢失了，这里重新写一下以前写过的小项目介绍~~

这是一个输入PID、作者ID、tag就可以自动跳转到pixiv官网的对应位置的纯静态网页，基于Vue2+Vuetify~~（上古技术栈）~~

虽然在电脑端我们可以在浏览器中很方便地编辑url，手动导航到对应页面。但是有一个问题：一般中国式英雄发车牌号都是在手机app上，而iOS pixiv官方app是不支持直接搜索pid或作者id的，同时移动端浏览器编辑url也没那么方便。因此我写了这么一个软件，虽然简单，但是能在一些“关键时刻”节省时间，避免坏了“兴致”

网站部署在vercel上，中国大陆访问在大多数情况下通畅

::github{repo="AkariRin/pixiv-router"}
