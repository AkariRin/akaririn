---
title: "微软Edge浏览器卸载教程"
published: 2025-03-17T14:30:46+08:00
description: "萝卜青菜各有所爱，浏览器亦是如此。很不幸，微软剥夺了我们喜欢其他浏览器Chrome的权利，让我们无法通过常规途径卸载Edge，但是我们仍然能通过修改注册表等奇技淫巧将Edge卸载"
tags: [Edge]
category: "教程"
image: ""
draft: false
comment: true
---

萝卜青菜各有所爱，浏览器亦是如此。很不幸，微软剥夺了我们喜欢其他浏览器:spoiler[Chrome]的权利，让我们无法通过常规途径卸载Edge，但是我们仍然能通过修改注册表等奇技淫巧将Edge卸载

不过鉴于修改注册表过于麻烦，因此我们将介绍如何使用第三方开源卸载工具一键卸载Edge

## 卸载Edge

::github{repo="ShadowWhisperer/Remove-MS-Edge"}

访问该GitHub项目，可以看到根目录下有三个可执行文件，作用分别如下：

- `Remove-Edge.exe`：只卸载Edge，保留webview组件
- `Remove-EdgeWeb.exe`：卸载Edge和webview组件
- `Remove-NoTerm.exe`：只卸载Edge，完全静默，适合用于计划任务等自动化场景，与本教程无关

下载对应可执行文件，右键选择`以管理员身份运行`即可，软件将会自动卸载Edge

卸载完成后即可删除该文件

## webview组件

webview组件是以下软件的依赖：

- ImageGlass
- Lenovo USB Recovery Creator Tool
- Microsoft照片(Edit)
- Quicken
- Windows邮件
- Xbox App

因此<span style="color: #ff0000;">不建议卸载webview</span>

## 注意事项

删除Edge可能会造成Windows系统更新失败的循环

解决方案：重新按照Edge后更新操作系统，更新完成后再次卸载Edge
