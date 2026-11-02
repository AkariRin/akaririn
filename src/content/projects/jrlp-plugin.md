---
title: "jrlp-plugin"
published: 2026-01-06
description: "麦麦插件：今日老婆"
status: "published"
tags:
  - "Python"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/AkariRin/jrlp-plugin"
---

🎲每天在群里随机抽取一位群友作为你的"群老婆"，每日限抽一次，结果当天有效。

## 触发命令

群内发送`抽老婆`、`今日老婆`、`jrlp`即可抽取

### 自定义触发命令

你可以通过修改配置文件中的 `command.regex` 来自定义触发命令的正则表达式：

```toml
[command]
# 示例1: 添加更多触发词
regex = '^(今日老婆|抽老婆|jrlp|今天老婆|我的老婆|抽个老婆)$'

# 示例2: 支持带前缀的命令（如 /、!、！）
regex = '^[/!！](https://github.com/AkariRin/jrlp-plugin/blob/main/今日老婆|抽老婆|jrlp)$'

# 示例3: 支持更灵活的匹配（包含"老婆"关键词即可触发）
regex = '.*老婆.*'
```

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
enabled = true             # 是否启用插件
config_version = "1.1.5"   # 配置版本

[napcat]
address = "napcat"         # napcat服务器连接地址
port = 3000                # napcat服务器端口

[command]
# 今日老婆命令的正则表达式，用于匹配触发命令的消息
# 默认支持: 今日老婆、抽老婆、jrlp
regex = '^(今日老婆|抽老婆|jrlp)$'

[messages]
# 已抽取老婆时的提示文本
# 支持占位符: {wife_name} 老婆昵称, {wife_qq} 老婆QQ号
already_rolled_text = "你今天已经有群老婆{wife_name}({wife_qq})了，要好好对待她哦~"

# 新抽取老婆时的提示文本
# 支持占位符: {wife_name} 老婆昵称, {wife_qq} 老婆QQ号
new_roll_text = "你今天的群老婆是:{wife_name}({wife_qq})"

[admin]
enabled = true             # 是否启用管理功能
userlist = []              # 有管理权限的用户QQ号列表，如 ["114514", "1919810"]
```

## 管理员命令

仅配置在 `admin.userlist` 中的用户可使用以下管理命令：

### 查询指定成员的老婆

```
/jrlp query <群成员QQ号>           # 群聊模式
/jrlp query <群号> <群成员QQ号>    # 私聊模式
```

查询指定群成员今日抽取的老婆

### 查询群今日所有老婆记录

```
/jrlp queryall [页码]              # 群聊模式
/jrlp queryall <群号> [页码]       # 私聊模式
```

分页查询群内今日所有抽取记录，每页显示10条

### 修改/指定老婆

```
/jrlp override <群成员QQ号> <老婆QQ号>           # 群聊模式
/jrlp override <群号> <群成员QQ号> <老婆QQ号>    # 私聊模式
```

强制修改或指定某成员今日的老婆。如已有记录则更新，否则新建记录

## 注意事项

- ⚠️ 该命令仅支持**群聊**环境，私聊无法使用
- 📍 抽取结果按群隔离，不同群的抽取互不影响
- 🔄 每天凌晨 0 点后可重新抽取
- 🚫 不会抽到自己

## 数据存储

插件数据存储在 `jrlp.db` SQLite 数据库
