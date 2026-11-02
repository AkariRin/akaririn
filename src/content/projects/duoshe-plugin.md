---
title: "duoshe-plugin"
published: 2026-01-07
description: "麦麦插件：夺舍"
status: "published"
tags:
  - "Python"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/AkariRin/duoshe-plugin"
---

定期随机选择活跃用户进行群名片交换的趣味插件

## 功能

- 每个群聊独立定时执行"夺舍"操作
- 基于24小时活跃度选择用户（使用指数分布算法）
- 拍一拍目标用户并交换群名片

## 配置napcat

在napcat的网络配置中添加一个HTTP服务器：

1. 打开napcat的配置界面（WebUI或配置文件）
2. 在"网络配置"中点击"添加"，选择"HTTP服务器"
3. 配置以下参数：
   - **主机地址**: `0.0.0.0`（允许外部访问）或 `127.0.0.1`（仅本机访问）
   - **端口**: `3000`（与插件配置中的 `napcat.port` 保持一致）
   - **启用CORS**: ✅ 开启
   - **Token**: 留空（不设置鉴权）
4. 保存配置并重启napcat

> ⚠️ 注意：如果napcat和插件不在同一台机器上，请确保防火墙放行对应端口。

## 配置说明

插件配置文件位于 `config.toml`：

```toml
[plugin]
enabled = true                 # 是否启用插件
config_version = "1.0.0"       # 配置文件版本

[napcat]
address = "napcat"             # Napcat服务器地址
port = 3000                    # Napcat服务器端口

[schedule]
min_interval = 360             # 最小间隔时间（分钟）
max_interval = 480             # 最大间隔时间（分钟）

[misc]
lambda_param = 1.5             # 指数分布的λ参数，值越大越倾向于选择活跃度高的用户
custom_cards = []              # 自定义名片列表，将附加到候选名片列表中
```

### `[misc]` 杂项配置

- **lambda_param**: 指数分布的λ参数，用于选择目标用户
  - 值越大：越倾向于选择活跃度高的用户
  - 值越小：选择更加均匀随机
  - 默认值：`1.5`

- **custom_cards**: 自定义名片列表
  - 类型：字符串数组
  - 默认值：`[]`（空列表）
  - 说明：当机器人是管理员时，会从候选名片中随机选择一个设置给目标用户。候选名片包括：
    1. 机器人的昵称（`bot.nickname`）
    2. 机器人的别名列表（`bot.alias_names`）
    3. 自定义名片列表（`custom_cards`）
  - 示例：
    ```toml
    custom_cards = ["小可爱", "群友A", "今天也要开心鸭"]
    ```


## 使用

1. 将插件目录放置在 `plugins/` 下
2. 调整 `config.toml` 配置
3. 重启麦麦
